---
layout: post
title: "Kotlin 'coercion to unit' 和 'UnitConversion'"
mathjax: true
tags: kotlin, language feature, coercion to unit, unitconversion
---
`apply`接受的函数参数类型为`T.() -> Unit`，`run` 接受的函数参数类型为`T.() -> R`,但是下面的代码并不会报错
```kotlin
data class TT(
    val id: String,
    val vv: String
)

typealias SS = List<TT>

fun SS.sortedBy(): List<TT> {
    return this.sortedBy { it.id }
}
fun main() {
    val ll = listOf(TT("ASD-a-04", ""), TT("ASD-a-03", ""))
    println(0)
    println(ll.sortedBy { it.id })
    println("ll: $ll")
    println(1)
    val f = SS::sortedBy
    println(f::class.jvmName)
    println(2)
    println(SS::sortedBy.returnType)
    println(3)
    println(ll.apply(SS::sortedBy))
    println("ll: $ll")
    println(4)
    println(ll.run(SS::sortedBy))
    println("ll: $ll")
}
```
上面的代码中，给`apply`和`run`传入了同一个参数，结果都没报错，但是`run`和`apply`的参数类型是不一样的？
这就涉及到Kotlin的语言特性: Unit coercion。
# Coercion to kotlin.Unit
[官方文档](https://kotlinlang.org/spec/statements.html#coercion-to-kotlin.unit)
大意是：在一个代码块的类型不是`Unit`而我们期望他的类型是`Unit`时，放松类型检查，忽略和`Unit`不匹配的类型。
官方文档里的例子:
```kotlin
fun foo() {
    val a /* : () -> Unit */ = {
        if (true) 42
        // CSB with no last expression
        // Type is defined to be `kotlin.Unit`
    }

    val b: () -> Unit = {
        if (true) 42 else -42
        // CSB with last expression of type `kotlin.Int`
        // Type is expected to be `kotlin.Unit`
        // Coercion to kotlin.Unit applied
    }
}
```
和上面的`SS::sortedBy`(function reference)一样，代码块的返回值不是`Unit`,但是放松了类型检查，没有报错。

# UnitConversion
此外,1.4版本还引入了一个新的特性`UnitConversion`, 这个特性默认关闭，下面的代码会报错:
```kotlin
val ll = listOf(TT("ASD-a-04", ""), TT("ASD-a-03", ""))
val f = SS:sortedBy
println(ll.apply(f)) // The feature "unit conversion" is disabled
```
在`build.gradle.kts`中加入编译器参数可以开启此特性:
```kotlin
val compileKotlin: KotlinCompile by tasks
compileKotlin.kotlinOptions {
    languageVersion = "1.4"
    freeCompilerArgs = listOf("-XXLanguage:+UnitConversion") // 开启 UnitConversion的编译器参数
}
```
开启`UnitConversion`之后，上面的代码就不再报错。

# callable reference

开头的代码中用了 `SS::sortedBy`，这是一个 [callable reference](https://kotlinlang.org/docs/reflection.html#callable-references)，并不是一个代码块，它也支持 coercion to unit。
因为java 8也支持这个特性。相关链接 : [YouTrack](https://youtrack.jetbrains.com/issue/KT-11723) , [jls 15.13.2](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.13.2), [StackOverflow answer](https://stackoverflow.com/questions/32539100/why-does-this-java-8-method-reference-compile/32539211#32539211)