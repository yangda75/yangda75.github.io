---
title: "Node.js 文档阅读：Timers in Node.js"
layout: post
tags: 
- node.js, 
- timers
---

# Table of Contents

1.  [从Node.js中的计时器说起](#orgc678b18)
2.  [使用Node.js控制时间](#orga240b35)
    1.  [“在我说的时候”执行～ `setTimeout()`](#org7183e97)
    2.  [“马上”执行～ `setImmediate()`](#orgfc16be1)
    3.  [“不停地”执行～ `setInterval()`](#org2d02720)
3.  [清除](#orgbb6d390)
4.  [丢弃 `Timeout`](#orga11bc28)
<https://nodejs.org/en/docs/guides/timers-in-node/>


<a id="orgc678b18"></a>

# 从Node.js中的计时器说起

Node.js中的计时器模块含有在指定的一段时间之后执行代码的函数。计时器不需要使用 `require(*)` 来导入，因为所有的方法都是全局可用的，这是为了模仿浏览器的JS API。如果想要深入了解计时器函数何时被执行，推荐阅读“事件循环”。


<a id="orga240b35"></a>

# 使用Node.js控制时间

Node.js的API提供好几种在一段时间后执行代码的方式。下面的函数可能看起来很眼熟，因为很多浏览器都支持这些函数，但是实际上Node.js提供它自己对这些函数的实现。计时器和系统的关系很紧密，虽然这些API和浏览器API长得很像，但是他们实际上有一些区别。


<a id="org7183e97"></a>

## “在我说的时候”执行～ `setTimeout()`

`setTimeout()` 可以用来规划在一段时间之后执行代码。这个函数和浏览器JS API中的 `window.setTimeout()` 很像。然而不能向它传一串包含代码的字符串。

`setTimeout()` 接受两个参数，第一个是要执行的函数，第二个是延迟，单位是毫秒。也可以给它传递更多的参数，剩下的参数会传入要执行的函数。
下面是一个例子：

{% highlight javascript %}
function myFunc(arg) {
  console.log(`arg was => ${arg}`);
}

setTimeout(myFunc, 1500, 'funky');
{% endhighlight %}

上面的 `myFunc()` 函数会在调用 `setTimout()` 之后大约1500毫秒执行。

这段时间并不是准确的，因为执行事件循环中阻塞的代码会延后执行的时间。唯一的保证是这段代码不会在给定的时间段到达之前执行。

`setTimeout()` 返回一个 `Timeout` 对象。这个返回的对象可以用来取消定时器( `clearTimeout()` )，也可以改变它的行为( `unref()` )


<a id="orgfc16be1"></a>

## “马上”执行～ `setImmediate()`

`setImmediate()` 会在当前event loop 循环执行完之后执行给定的代码。代码会在这一次事件循环的所有I/O执行完之后，下一次事件循环的所有定时器之前执行。这种执行方式可以看作“马上”执行，意思是 `setImmediate()` 之后的代码会在参数函数之前执行。

`setImmediate()` 的第一个参数是要执行的函数。剩下的所有参数都会传入参数。一个例子：

{% highlight javascript %}

console.log('before immediate');

setImmediate((arg) => {
  console.log(`executing immediate: ${arg}`);
}, 'so immediate');

console.log('after immediate');
{% endhighlight %}

传入 `setImmediate()` 的函数会在所有可以执行的代码被执行之后执行。输出是下面这样：

    before immediate
    after immediate
    executing immediate: so immediate

`setImmediate()` 返回一个 `Immediate` 对象，可以用来取消( `clearImmediate()` )

注意：不要把 `setImmediate` 和 `process.nextTick` 搞混了。他们很不一样。首先 `process.nextTick` 和计划好的（这一次事件循环）I/O操作一样会在所有的 `setImmediate` 之前执行。其次， `process.nextTick` 取消不了。


<a id="org2d02720"></a>

## “不停地”执行～ `setInterval()`

如果需要重复执行一段代码，可以用 `setInterval()` 。 `setInterval()` 的第一个参数是要执行无数次的函数，第二个参数是两次执行间的间隔。
和 `setTimeout()` 一样，剩下的参数会传入要执行的函数。和 `setTimeout()` 一样，时间间隔可能延长。

{% highlight javascript %}

function intervalFunc() {
  console.log('Cant stop me now!');
}

setInterval(intervalFunc, 1500);
{% endhighlight %}

上面的例子中， `intervalFunc()` 每过大约1500毫秒执行一次。
和 `setTimeout` 一样， `setInterval()` 返回一个 `Timeout` 对象。


<a id="orgbb6d390"></a>

# 清除

想要取消一个 `Timout` 或者 `Immediate` 应该怎么做？ `setTimout` , `setInterval`, `setImmediate` 返回可以用来引用设定的 `Timeout` 或者 `Immediate` 对象的计时器对象。通过将这些对象传入对应的 `clear` 函数，就能够彻底取消执行。对应的函数是 `clearTimeout()`, `clearImmediate()`, `clearInterval()` 。

{% highlight javascript %}

const timeoutObj = setTimeout(() => {
  console.log('timeout beyond time');
}, 1500);

const immediateObj = setImmediate(() => {
  console.log('immediately executing immediate');
});

const intervalObj = setInterval(() => {
  console.log('interviewing the interval');
}, 500);

clearTimeout(timeoutObj);
clearImmediate(immediateObj);
clearInterval(intervalObj);
{% endhighlight %}


<a id="orga11bc28"></a>

# 丢弃 `Timeout`

`setTimeout` 和 `setInterval` 都会返回 `setTimout` 对象。可以使用 `ref` 和 `unref` 方法来修改 `Timeout` 的行为。如果使用 `set` 方法设置了一个 `Timeout` ，使用 `unref` 会修改它的行为：如果 `Timeout` 是最后要执行的代码, `unref` 之后就不执行了。 `ref` 可以取消 `unref()` ，但是出于性能考虑(JS 里好像语义一致性并没不重要)，并不能完全恢复，

{% highlight javascript %}

const timerObj = setTimeout(() => {
  console.log('will i run?');
});

// if left alone, this statement will keep the above
// timeout from running, since the timeout will be the only
// thing keeping the program from exiting
timerObj.unref();

// we can bring it back to life by calling ref() inside
// an immediate
setImmediate(() => {
  timerObj.ref();
});
{% endhighlight %}
