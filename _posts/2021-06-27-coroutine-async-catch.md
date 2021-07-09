---
layout: post
title: "Kotlin Coroutine debug 记录"
mathjax: true
tags: kotlin, coroutine, bug
---

在客户现场出现了程序关键流程“死掉”的问题，指挥叫我交TP到现场调bug。

看了半天log，发现是有个异常传到顶层导致 coroutine scope 销毁了。
原因是调 `async` 的地方没加 `try-catch` 捕获异常。

现场调完回上海，去机场的路上很菜地封了个方法，
用两张机票换来的十行代码：
```kotlin
fun <T> CoroutineScope.wrappedAsync(f: suspend CoroutineScope.() -> T): Deferred<T?> {
    return this.async {
        try {
            f()
        } catch (e: Exception) {
            logger.error("Exception in async job")
            null
        }
    }
}
```

已被自己菜哭。