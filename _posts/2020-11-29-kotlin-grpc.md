---
layout: post
title: "Kotlin gRPC简单使用总结"
tags: kotlin, gRPC, coroutine
---

# gRPC
选择使用gRPC的原因是它支持的语言很多，便于客户在我们的系统上进行二次开发。

关于gRPC的简介此处不再重复，参考官网https://grpc.io/ 。

# build.gradle.kts
gRPC使用protocol buffer文件(.proto后缀)做接口定义文件。在.proto文件中写好接口定义后，需要使用protoc生成java代码，
再根据生成的java代码生成kotlin代码。

需要在编译时生成代码，所以gradle.build.kts需要使用 `protobuf` 插件。

需要在写代码时，让ide能够为生成的代码建立索引，从而提供补全提示和语法检查，需要将生成的代码加入项目的源文件列表，所以需要使用 `idea` 插件，我用的是IDEA这个ide，如果使用其他ide，可能有不同。

此外，需要在生成代码前删掉之前生成的代码，以免产生不应该出现的ide错误提示。由于目前生成代码的gradle插件没有自动删除，我们需要在build.gradle.kts中指示它去删除。

gRPC的官方文档比较简单，很多细节都需要自己去看代码，去实验才能找到正确的做法，下面这份代码也是我参考了很多GitHub上的例子和StackOverflow上的讨论，根据自己的需要写出来的，花了整整半天时间。希望以后gRPC-kotlin能给个详细点的文档，protobuf这种需要生成代码并且工程依赖生成的代码的库集成起来还是有点费事的。

```kotlin
import com.google.protobuf.gradle.*

plugins {
    id("com.google.protobuf") version "0.8.13"
    idea
}

val protobufVersion = "3.11.0"
val grpcKotlinVersion = "0.1.2"
val grpcVersion = "1.29.0"

// 需要修改注册表才能用protoc-gen-grpc-kotlin来生成kt文件
// https://github.com/grpc/grpc-kotlin/issues/81
dependencies {
    implementation("io.grpc:grpc-netty-shaded:$grpcVersion")
    implementation("io.grpc:grpc-protobuf:$grpcVersion")
    implementation("io.grpc:grpc-stub:$grpcVersion")
    implementation("io.grpc:grpc-kotlin-stub:$grpcKotlinVersion")
    implementation("io.grpc:grpc-services:$grpcVersion")
    compileOnly("javax.annotation:javax.annotation-api:1.2")
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:$protobufVersion"
    }
    plugins {
        id("grpc"){
            artifact = "io.grpc:protoc-gen-grpc-java:$grpcVersion"
        }
        id("grpckt") {
            artifact = "io.grpc:protoc-gen-grpc-kotlin:$grpcKotlinVersion"
        }
    }
    generateProtoTasks {
        // 生成新的之前先删除旧的
        all().forEach {
            it.doFirst {
                delete(it.outputs)
            }
        }
    
        all().forEach {
            it.plugins {
                id("grpc")
                id("grpckt")
            }
        }
    }
    // 指定生成的代码放在哪个文件夹下
    generatedFilesBaseDir = "$projectDir/src/generated"
}

// 将生成代码所在文件夹加入项目代码目录列表
idea{
    module {
        sourceDirs.plusAssign(file("$projectDir/src/generated/"))
    }
}
```

# 协程
gRPC-kotlin 支持协程。生成的服务器和客户端(stub)都有协程版本。如果proto文件中，你定义了 `java_outer_classname=DoorProto` 和 `service Door` ，那么生成的代码中就会有一个文件，名字叫 `DoorProtoKt.kt` ，其中有一个对象， `DoorProtoKt` 。

实现协程服务器时，需要扩展 `DoorProtoKt.DoorCoroutineImplBase()` 类；

创建协程客户端时，需要使用 `DoorProtoKt.DoorCoroutineStub()` 类。

gRPC的方法有四种通信模式，unary call，server streaming，client streaming，bidirectional streaming。

unary call模式的方法为suspend函数，
streaming类的方法中，stream 为kotlin中的 `Flow` ，服务器返回stream的方法不是suspend函数，不返回stream的是suspend函数。

既然gRPC支持结构化并发(structured concurrency)，那我也可以用结构化并发的写法来写gRPC相关的代码。

# 创建CoroutineScope

首先，gRPC是整个软件中的一个模块，有自己的生命周期，先给它创建一个自己的 `CoroutineScope` 跟gRPC相关的任务都在这个scope中执行。 
```kotlin
    private val exceptionHandler = CoroutineExceptionHandler { _, e ->
        logger.error("Error in gRPC scope:", e)
    }
    
    private val scope: CoroutineScope = CoroutineScope(Job() + Dispatchers.IO + exceptionHandler)
```

# 服务器的创建、启动和关闭

gRPC官方给的例子中，使用了线程池，配合 `blockUntilShutdown()` 方法，阻塞线程，直到服务器关闭。我们可以用 `kotlinx.coroutines.channels.Channel` 配合一个关闭服务器的信号来代替它，达到阻塞的效果，还可以在收到关闭服务器的信号时，做更多处理。

```kotlin
 // 接收关闭服务器信号的channel
    private val serverSignalChannels: MutableMap<String, Channel<GrpcServerSignal>> = ConcurrentHashMap()
// 服务器对应的Job
    private val serverJobs: MutableMap<String, Job> = ConcurrentHashMap()
    
    fun startServer(port: Int, services: List<BindableService>, serverName: String) {
        serverSignalChannels[serverName] = Channel()
        serverJobs[serverName] = scope.launch(Dispatchers.IO) {
            logger.info("building server $serverName")
            val builder = ServerBuilder.forPort(port)
            services.forEach {
                builder.addService(it)
            }
            val server = builder.build()
            server.start() // 开始服务，这个方法不是阻塞的，需要我们自己阻塞线程，不然会立即退出。
            logger.info("${serverName}server started@$port")
            // 阻塞，等待关闭服务器的信号
            for (signal in serverSignalChannels[serverName]!!) {
                logger.info("receive server signal: $signal")
                when (signal) {
                    GrpcServerSignal.Shutdown -> {
                        // 收到关闭服务器信号了，进行处理
                        server.shutdown().awaitTermination()
                        logger.info("server $serverName terminated: ${server.isTerminated}")
                    }
                    GrpcServerSignal.ShutdownNow -> {
                        server.shutdownNow().awaitTermination()
                        logger.info("server $serverName terminated: ${server.isTerminated}")
                    }
                }
            }
            logger.info("$serverName Job done")
        }
    }
    
    fun stopServer(serverName: String) {
        if (serverSignalChannels.containsKey(serverName)) {
            logger.info("stopping server $serverName")
            runBlocking { // 发起关闭服务器请求的线程需要阻塞，等待服务器关闭完成
                val channel = serverSignalChannels[serverName]!!
                channel.send(GrpcServerSignal.Shutdown)
                channel.close() // 发送信号之后关闭channel，停止阻塞
                serverSignalChannels.remove(serverName)
                serverJobs[serverName]?.join() // 使用Job.join()来等待服务器关闭完成
                serverJobs.remove(serverName)
            }
        }
    }
```

# CoroutineScope的异常处理: try-catch + CoroutineExceptionHandler
CoroutineScope是可以嵌套的，如果异常被捕获了，那么和外层scope没关系，如果异常没有被捕获，那么就会层层上报，直到有一层的CoroutineExceptionHandler捕获这个异常，一个scope的CoroutineExceptionHandler捕获到异常时，这个scope就结束了，再用 `.isActive()` 方法，会返回false。

```kotlin
    fun grpcScope(f: suspend () -> Unit) {
        scope.launch {
            try {
                f()
            } catch (e: Exception) { // do not propagate to parent scope
                logger.error("Exception occurred in grpcScope", e)
            }
        }
    }
```
上面这段代码中，用try-catch包围传入的函数，避免这个函数的异常杀掉整个scope。
