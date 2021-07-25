---
layout: post
title: "Kotlin scope function"
mathjax: true
tags: kotlin, scope function
---

官方文档 : [scope function](https://kotlinlang.org/docs/scope-functions.html)

## 概念
概念是最重要的，学习中最重要的任务是掌握概念。（忘记在哪里看到的了，很有道理）

### scope function:
<!-- 官方文档:

The Kotlin standard library contains several functions whose sole purpose is to execute a block of code within the context of an object. When you call such a function on an object with a lambda expression provided, it forms a temporary scope. In this scope, you can access the object without its name. Such functions are called *scope functions*. There are five of them: `let`, `run`, `with`,`apply` and `also`.

Basically, these functions do the same: execute a block of code on an object. -->
Kotlin标准库中，(1.4版本)具体是`kotlin-stdlib-1.4.0.jar`中的`Standard.kt`文件中定义的一组函数，它们都接受一个lambda函数作为参数，对一个对象调用这些方法，会构成一个临时的域（scope），在这个域中的代码可以不用变量名来指代对象。这种函数就叫 scope function。
scope function主要的作用就是对一个对象执行一块代码。
### context object:
scope function创建的域中的上下文对象（用`it`或者`this`指代的对象）就是上下文对象。例如，
```kotlin
something?.let{
    println("something: ${it.name}") // it = something
    name = it.name // it = something
}
```
上面代码块中的`let`调用作用于`something`，创建的域中，`it`就是`something`，也就是“上下文对象”。
### receiver:
kotlin 的官方文档中没找到专门解释这个概念的文档，StackOverflow上有个帖子很好: [SO帖子](https://stackoverflow.com/a/45875492/14547011)

receiver的定义：Kotlin中，函数可以有一个(SO帖子里写的是多个，通过多条定义得到，但是我理解Kotlin的函数类型限制了receiver的数量，因为receiver直接写在函数类型中，且只能写一个)receiver，在函数的代码中可以直接(原文叫without qualifying,不知道怎么翻译)获取receiver类型的属性。
Kotlin的函数类型为 `receiver.(parameter list) -> return type`, [Kotlin文档](https://kotlinlang.org/docs/lambdas.html#function-types)
```kotlin
val minus100: Int.() -> String = {minus(100).toString()}
```
上面的代码块中，`minus100`是一个函数，它的类型是`Int.() -> String`，所以receiver是`Int`，在函数代码中，直接调用了`Int`类型的`minus`方法，没有加`it`，或者说省略了`this`，这块代码仿佛就在`Int`类的定义里一样，所以叫extension function，扩展函数，意思是仿佛扩展了一个类。

从某种程度上，receiver将函数内代码指代上下文的方式从`it`变成了`this`，而且`this`可以省略。在嵌套调用lambda函数时，用`this`避免了内层`it`覆盖外层`it`的问题，但是增加了`this`到底指向谁的的问题。
例如:
```kotlin
val minus10it: (Int) -> Int = { it.minus(10) }
val minus10this: Int.() -> Int = { minus(10) }

val a = 10000.let {
    minus10it(it).let {
        minus10it(it)
    }
}

val b = 10000.run {
    minus10this().run {
        minus10this()
    }
}
```
用`it`的版本里，出现了内外层`it`打架的问题，用`this`就没有这个问题。

## let
`let`的定义:
```kotlin
public inline fun <T, R> T.let(block: (T) -> R): R {
    return block(this)
}
```

`let`的函数签名中出现了两个类型参数 `T` `R`: `T`是receiver的类型，`R`是返回值的类型。`let`返回的是将上下文对象作为参数传给block得到的结果。
实际开发中，用的最多的是 `?.let`这种模式，如：
```kotlin
resource?.let{
    // ...
} 
```
在不为`null`时进行操作。

还有一个常用场景是代替临时变量，比如有一个表达式的结果需要在后续操作中用到，可以用`let`来避免创建临时变量，例如:
```kotlin
// 临时变量
val tmp = complexAndExpensiveOperation()
doA(tmp)
doB(tmp)
// let
complexAndExpensiveOperation().let{
    doA(it)
    doB(it)
}
```

## run
`run`的定义：
```kotlin
public inline fun <T, R> T.run(block: T.() -> R): R {
    return block()
}
```
和`let`的区别：`run`中block里用`this`指代上下文对象，`let`中用`it`指代。
`run`还有另一个签名:
```kotlin
public inline fun <R> run(block: () -> R): R {
    return block()
}
```
这个版本没有上下文变量，没怎么用过。

## apply
`apply`的定义:
```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T {
    block()
    return this
}
```
执行一个函数，并返回上下文对象。纯粹为了函数的副作用。

## also
`also`的定义:
```kotlin
public inline fun <T> T.also(block: (T) -> Unit): T {
    block(this)
    return this
}
```
和`apply`的区别是block中用`it`指代上下文对象。
刚学Kotlin的时候，有个用`also`交换两数的例子：
```kotlin
var a = 123
var b = 321

a = b.also { b = a }
println(a) // 321
println(b) // 123
```
在执行`also`的传入参数block时，上下文对象被拷贝了，并且在`{b=a}`这个代码块中，并没有改变上下文对象`it`
```kotlin
var a = 123
var b = 321

a = b.also {
    println("b=$b, it=$it") // b=321, it=321
    b = a
    println("b=$b, it=$it") // b=123, it=321
}
println(a) // 321
println(b) // 123
```
`b`先是被拷贝到`also`中的`this`,然后作为block里的`it`，执行完block后，`also`返回了`this`，所以，`b.also{b=a}`的值是一开始的`b`，而现在的`b`在执行block的时候变成了一开始的`a`。

## with
`with`的定义：
```kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    return receiver.block()
}
```
和`run`的第一种定义比较像，都是有一个receiver，一个带receiver的lambda，不同的是receiver的位置，`run`是`receiver.run(block)`，`with`是`with(receiver, block)`，Kotlin 语言规范中说明了两种类型的区别：[Function types](https://kotlinlang.org/spec/type-system.html#function-types)。`run` 为`FTR(RT,A1)->R`,`with`为`FT(RT, A1)->R`，或者说，`run`为`T.(T.()->R)->R`，`with`为`(T,T.()->R)->R`。这两种定义在重载时有区别。`run`可以作为很多类型的扩展函数，`with`并不是扩展函数。