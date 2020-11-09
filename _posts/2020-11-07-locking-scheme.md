---
layout: post
title: "记一次并发问题"
tags: kolin,java,lock
---
先上代码
```kotlin
class AioTcpClient{
    // ......
    private val lock = Mutex()
    var flowNo = 0
    
    private val nextFlowNo: Int
        @Synchronized
        get() {
            flowNo = (flowNo + 1) % 65535
            return flowNo
        }
    
    // ......
    suspend fun request(msgNo: Int, msg: String): String {
        // ......
        reqBuffer.putShort(nextFlowNo.toShort())
        // ......
        // send and read response
        return lock.withLock {
            AioTcpHelper.writeAll(channel, reqBuffer)
            readResponse(flowNo)
        }
    }
    
    suspend fun readResponse(expectedFlowNo: Int?): String {
        // ......
        val actualFlowNo = headerBuffer.short.toInt()
        
        if (actualFlowNo != expectedFlowNo && expectedFlowNo != null) {
            throw RuntimeException("FlowNo mismatch, expected: $expectedFlowNo, got: $actualFlowNo")
        }
        // ......
    }
    // ......
}
```
这是一段Kotlin代码（跑在JVM上），用java的异步io实现一个tcp客户端。
截取的部分有三个属性和三个方法：
三个属性：
1. `lock`，一把锁
2. `flowNo`，当前流水号，如果发送的流水号和读取的流水号不同，意思就是除了问题，得有人负责
3. `nextFlowNo`，下一个流水号

三个方法：
1. `nextFlowNo`的`get()`（访问器）方法，用于获取新的`flowNo`。
2. `request`，发送请求并读取回复，用`lock`这把锁将
3. `readResponse`，读取回复

在写`nextFlowNo`的访问器的时候用的原生锁，kotlin中的`@Synchronized`标记类的作用就是将生成的java方法用`synchronized`关键字修饰。在写`request`的方法的时候我没法用原生锁，因为
不能在`synchronized`关键字标记的代码块中进行线程休眠，这和`suspend`冲突。用`kotlinx.couroutines.sync.Mutex`才合适，于是我就用了。

但是，运行的时候出了问题，贴一段log
```
......
req: [577]
FlowNo mismatch, expected: 578, got: 577
......
```
这两行是紧挨在一起的。第一行是发送了个请求，流水号为577，但是接收回复的时候，却报错了，流水号不符，期望流水号为578，从tcp字节流中得到的是577，出问题了。

还没发578流水号对应的包呢，怎么就expected: 578了？

。。。（表示看了半小时代码，一个句号表示10分钟，流水账要记全）

问题就在之前贴出来的代码中？你看出来了吗？

问题就在锁上面。更新流水号和发送请求读取回复的两个过程用了两把锁。在请求组包完成之后收到回复之前，
其他线程都没法发请求，但是是可以执行`nextFlowNo`的访问器方法的，因为这个访问器方法是用原生锁来锁的，而发送请求读取回复的过程是用`lock`来限制并发的。
所以才会出现收到请求之前`flowNo`被其他线程改掉的问题。怎么解决呢？

把`flowNo`改成 volatile 变量没用，因为tcp服务器发回来的流水号还是改之前的

把`lock`改成原生锁，不行，因为`suspend`

把原生锁改成`lock`，可以，问题解决。

此外，kotlin现在好像还不支持把访问器写成suspend函数，难受。

还好这次的并发问题比较好重现，不然真的很难发现是自己锁的不对。在每次写并发代码的时候要先看看jcip的前几章，还是有道理的。
