---
title: "Node.js 文档阅读笔记：Don't Block the Event-Loop"
date: <span class="timestamp-wrapper"><span class="timestamp">&lt;2019-12-30 Mon&gt;</span></span>
layout: post
categories: 
tags: 
- Node.js, 
- note, 
- blocking, 
- event-loop
---

# Table of Contents

1.  [总结](#orgda843d2)
2.  [为什么不应该阻塞事件循环和Worker Pool?](#orgf7152fd)
3.  [快速回顾Node](#org1d77d96)
    1.  [什么代码在事件循环中执行？](#org1018ebe)
    2.  [什么代码在Worker Pool 上执行？](#org802f0b7)
    3.  [Node怎么确认接下来运行什么代码？](#org6a400ee)
    4.  [这对应用设计意味着什么？](#org3dd5847)
4.  [不要阻塞事件循环](#org7a4d425)
    1.  [应该小心到什么程度？](#org1124d42)
    2.  [阻塞事件循环：REDOS](#org9a91e83)
    3.  [阻塞事件循环：Node核心模块](#org376f5fb)
    4.  [阻塞时间循环: JSON DOS](#org4676ea2)
    5.  [在不阻塞事件循环的前提下进行复杂计算](#org807db24)
        1.  [分区,partitioning](#org4c4e178)
        2.  [分流,offloading](#org4a8d23b)
5.  [不要阻塞Worker Pool](#org3f84085)
    1.  [最小化任务完成时间差异](#org059b649)
    2.  [任务分区](#org057fbff)
    3.  [避免任务分区](#orgcd202f3)
    4.  [Worker Pool :结论](#org8191d44)
6.  [NPM 包的风险](#orgd769cfa)
7.  [总结](#org49e271f)
<https://nodejs.org/en/docs/guides/dont-block-the-event-loop/>

涉及到不同操作系统时，本文档以linux为主。


<a id="orgda843d2"></a>

# 总结

Node.js 在事件循环中（初始化和回调函数）执行js代码，提供一个Worker Pool来处理昂贵的操作，比如文件I/O。Node可以很好地扩大规模，有时比重型的Apache还好。Node规模化能力的关键是它使用了很少的线程就能处理很多客户端。如果Node可以用更少的线程完成任务，那么它就可以讲更多的系统时间和内存用在处理客户端的需求而不是处理线程的固定开销（overheads该怎么翻译？），这个固定开销包括内存处理和上下文切换。但是因为node只有很少的几个线程，你的应用必须要有一个合适的结构才能聪明的使用它们。

这是一个让你的Node服务器速度很快的经验法则：当任何时候每个客户端关联的任务(work)都“小”的时候，Node很快。

这对于事件循环中的回调和Worker Pool中的任务适用。


<a id="orgf7152fd"></a>

# 为什么不应该阻塞事件循环和Worker Pool?

Node使用数量很少的线程来处理很多客户端的请求。在Node中有两种线程：一个事件循环（也叫做主循环，主线程，事件线程等），和一个含有k个Woker的Worker Pool（也叫线程池）。

如果一个线程使用很长的时间来执行一个回调函数（事件循环）或者任务（Worker），我们称它“被阻塞了”。当一个线程在为一个客户端工作时被阻塞了，那么他就不能处理其他客户端的请求了。这就提供了两个防止时间循环或者Worker Pool阻塞的动力：

1.  性能：如果你经常在这两种线程中的一种上进行重型的操作， *吞吐量(请求\\/秒)* 就会低。
2.  安全性：如果有一种输出可以使得一个线程阻塞，那么一个恶意的客户端就可以提交这个“坏输入”，来阻塞线程，使得其他客户无法得到服务。这将是一个拒绝访问攻击。


<a id="org1d77d96"></a>

# 快速回顾Node

Node使用一个时间驱动的架构：他有一个事件循环（for orchestration 不知道怎么翻译），Worker Pool用来处理昂贵的操作。


<a id="org1018ebe"></a>

## 什么代码在事件循环中执行？

在启动的时候，Node应用首先完成一个初始化过程，引用模块并且为事件注册回调函数。然后Node应用进入事件循环，通过调用合适的回调函数来响应客户端的请求。这个回调函数是同步执行的，并且有可能注册异步的请求在它退出后执行。这些异步请求的回调函数也会在事件循环上执行。

事件循环还会完成它的回调发起的非阻塞异步请求，比如网络I/O

总而言之，事件循环执行为了事件注册的JavaScript回调函数，并且负责完成异步非阻塞的请求，比如网络I/O。


<a id="org802f0b7"></a>

## 什么代码在Worker Pool 上执行？

Node的Worker Pool使用libuv实现，暴露了一个任务提交API。

Node 使用Worker Pool 来处理昂贵的任务。包括一些没有非阻塞版本的I\\/O操作，和一些CPU密集的任务。

这是使用Worker Pool的Node模块的API:

1.  I/O密集型：
    1.  DNS： `dns.loopup()`, `dns.lookupService()`
    2.  文件系统：所有的文件系统API，除了 `fs.FSWatcher()` 和其他使用libuv的线程池的明确的同步的
2.  CPU密集型：
    1.  Crypto: `crypto.pbkdf2()`, `crypto.scrypt()`, `crypto.randomBytes()`, `crypto.randomFill()`, `crypto.generateKeyPair()`
    2.  Zlib：所有的Zlib API，除了那些使用libuv的线程池的明确同步的

在很多的Node应用中，这些API是WorkerPool 任务的全部来源。使用c++插件的应用和模块可以提交其他的任务到Worker Pool。

为了完整性，我们此处说明，当你在事件循环的一个回调函数中调用这些API的时候，事件循环在进入Node的c++ binding来提交任务的时候会有一些时间花费。和整个任务的花费来比较，这些花费可以忽略不计，这也是为什么不在事件循环上进行这些操作的原因。当提交一个任务到WorkerPool的时候，Node提供一个指向Node C++ binding中对应的c++函数的指针。


<a id="org6a400ee"></a>

## Node怎么确认接下来运行什么代码？

抽象地说，事件循环和WorkerPool维护待执行的事件和任务。

实际上，事件循环并没有真的维护一个队列。相反，它有一个文件描述符集合，它要求操作系统监视这个集合，使用epoll(linux), kqueue(OSX),event ports(Solaris), IOCP(windows)这样的机制。这些文件描述符对应网络套接字或者它关注的文件，还有其他。当操作系统说这些文件描述符中的一个已经准备好了，事件循环就将它翻译成合适的事件，触发和事件关联的回调函数。

相对的，Worker Pool使用一个真实的队列，其中的元素是要执行的任务。一个Worker Pool 从它的任务队列中出队一个任务然后执行，然后，在它执行完成之后，抛出“至少完成了一个任务”的事件到事件循环。


<a id="org3dd5847"></a>

## 这对应用设计意味着什么？

在一个像Apache这样的一客户端一线程的系统中，每个在等待的客户端都分配了一个线程。如果一个处理客户端的线程阻塞了，操作系统会中断然后给另一个线程使用。操作系统保证需要较少时间的客户端不会被需要更多工作的客户端延误。

因为Node使用很少的几个线程处理大量客户端的请求，如果一个线程在处理一个客户端请求的时候阻塞了，那么剩余的客户端可能不会在线程完成回调函数或者任务之前得到处理。你的应用有责任公平对待所有的客户端。这意味着你不应该在单个回调函数或者任务中为了单个的客户端做太多工作。

这是为什么node容易扩展的一个原因，但是这也意味着你有责任确保公平的分配。下一章讨论如何保证事件循环和Worker Pool 的公平分配。


<a id="org7a4d425"></a>

# 不要阻塞事件循环

事件循环会注意每个新的客户端连接并且负责生成一个回复。所有进来的请求和出去的回复都通过事件循环。这意味着如果事件循环在任何一个地方花费过长的时间，所有当前和未来的客户端都得不到响应。

你应当保证永远不要阻塞事件循环。换句话说，你的每个JS回调都应该很快完成。这个当然也适用于你的 `await` ， `Promise.then` 和其他。

一个保证不阻塞的好办法就是估计回调函数的计算复杂度。如果你的回调函数无论参数是怎样都只要相同的运算规模，你总是会给每个在等待的客户端一次机会。如果参数不同会导致计算步骤数目不同，那么你应该考虑可能的参数。

一个执行时间不变的回调函数：

{% highlight javascript %}
app.get('/constant-time',(req,res) => {
    res.sendStatus(200);
});
{% endhighlight %}

一个O(n) 的回调：

{% highlight javascript %}
app.get('/counToN',(req,res) => {
    let n = req.query.n

    for (let i = 0; i < n; i++){
	console.log(`Iter {$i}`);
    }
});
{% endhighlight %}

一个O(n<sup>2</sup>) 的回调：

{% highlight javascript %}
app.get('countToN2', (req, res) => {
    let n = req.query.n;

    for (let i = 0; i < n; i++){
	for (let j = 0; j < n; j++){
	    console.log(`Iter ${i}.${j}`);
	}
    }

    res.sendStatus(200);
});
{% endhighlight %}


<a id="org1124d42"></a>

## 应该小心到什么程度？

Node使用Google V8引擎来执行JS, 这对于很多常见的操作是够用的。特例是正则表达式和JSON操作，下面会讨论。
然而，对于复杂的任务，你应当考虑限制输入长度并且拒绝过长的输入。这样的话，即使你的回调复杂度很高，运行时间也比较短。


<a id="org9a91e83"></a>

## 阻塞事件循环：REDOS

一个简单的阻塞事件循环的办法就是使用一个有弱点的正则表达式。
避免有弱点的正则表达式。

一个正则表达式使用一个规则来匹配输入的字符串。我们通常认为匹配过程的复杂度是 O(n) 。在很多时候，确实只需要遍历一次。不幸的是，有些时候正则表达式匹配需要指数级的遍历次数，也就是时间复杂度为 O(2<sup>n</sup>)。

   一个有弱点的正则表达式就是你的匹配引擎在匹配时需要遍历指数次的。这种表达式构成 REDOS(Regular Expression Denial Of Service)攻击。
怎么分辨一个有弱点的，也就是需要指数时间处理的正则表达式？这个问题很难回答，而且根据使用语言的不同而变化。这里有一些经验法则：

-   避免使用嵌套的修饰器，比如 `(a+)*` 。Node的正则表达式引擎可以很快地处理一些这样的表达式，另一些是指数时间的。
-   避免在交叠语句中使用或操作，比如 `(a|a)*` 。同样，这个只是有些时候很快。
-   避免使用后向引用，比如 `(a.*) \1.` 。没有一个正则表达式引擎保证在指数时间之内执行完这个。
-   如果你在做一些简单的字符串匹配，使用 `indexOf` 或者本地的等价方法，开销更小，永远不会超过 O(n)。
    
    如果你不确定你的表达式是不是有弱点，记住，Node通常不会出现有匹配但找不出的情况，即使是一个很长的字符串和一个有弱点的表达式。

当有一个不匹配的时候，指数行为就触发了，但是直到遍历过很多遍输入字符串之前，Node都不确定这件事。

一个 REDOS 的例子:

{% highlight javascript %}
app.get('/redos-me',(req, res) => {
    let filePath = req.query.filePath;

    // REDOS
    if (filePath.match(/(\/.+)+$/)){
	console.log('valid path');
    }
    else{
	console.log('invalid path');
    }

    res.sendStatus(200);
})
{% endhighlight %}

例子里面的表达式违反了规则1,使用了嵌套修饰器。

反REDOS资源

有一些检查正则表达式安全性的工具，比如：

-   safe-regex
-   rxxr2

但是，这些工具都不会检查出有弱点的正则表达式。


<a id="org376f5fb"></a>

## 阻塞事件循环：Node核心模块

一些Node的核心模块有同步的API，比如：

-   加密
-   压缩
-   文件系统
-   子进程
    这些API是很昂贵的，因为他们包含了大量的计算，需要I/O，或者两者都需要。这些API是为了写脚本方便而设计的，并不是为了在服务器端使用而设计的。如果你在事件循环中执行他们，他们会比普通的js指令执行更长的时间，导致事件循环阻塞。

在服务器中，你不应使用这些同步API:

-   加密 
    -   crypto.randomBytes
    -   crypto.randomFillSync
    -   crypto.pbkd2fSync
-   压缩
    -   zlib.inflateSync
    -   zlib.deflateSync
-   文件系统
    不要使用同步API
-   子进程
    -   `child_process.spawnSync`
    -   `child_process.execSync`
    -   `child_process.execFileSync`


<a id="org4676ea2"></a>

## 阻塞时间循环: JSON DOS

JSON.parse 和 JSON.stringify 是另外两个可能很昂贵的操作。他们复杂度是O(n)，在输入很长时需要很长时间。

如果你的服务器需要处理JSON对象，尤其是从客户端来的那些，在事件循环中处理这些字符串对象的时候你应该小心他们的长度。

例子，JSON阻塞。我们创造一个 2<sup>21</sup> byte的 `obj` 对象，然后对它 `JSON.stringify` ，再运行 `indexOf` ，最后 `JSON.parse` 。需要0.7秒来序列化，需要0.03秒来运行 `indexOf` ，总共大约1.3秒，字符串的长度是50M。


<a id="org807db24"></a>

## 在不阻塞事件循环的前提下进行复杂计算

如果你想要在不阻塞事件循环的同时进行复杂计算，你有两个选项，分区和分流(partitioning, offloading)。


<a id="org4c4e178"></a>

### 分区,partitioning

你可以将你的计算分区，每个分区都在事件循环中进行，并且正常将控制权转移给其他事件。在JS中很容易使用闭包来存储正在执行的任务的状态。


<a id="org4a8d23b"></a>

### 分流,offloading

如果你需要做一些更复杂的事情，分区并不是一个好的选择。这是因为分区只使用事件循环，但是你不能使用多个处理器核心。记住，事件循环应该组织处理客户请求，而不是亲自完成它们。对于一个复杂的任务，将负载从事件循环移动到WorkerPool中。

如何分流

你有两种选择：

1.  开发c++插件来使用Node.js内建的Node Worker Pool。
2.  你可以创建和管理自己的Worker Pool，最简单的方式是使用子进程和cluster。

你不应该为每个客户创建一个子进程。

分流的坏处

分流这个方法的坏处是它产生了交流成本。只有事件循环可以看到js应用的命名空间。对一个Worker，你不能在事件循环的命名空间中操作一个JS对象。相反的，你必须要序列化再反序列化对象，才能在Worker中使用这个对象的拷贝。

分流的一些建议

你也许想要区分CPU密集和I/O密集型的任务，因为他们有很不同的特性。

一个 CPU密集的任务，只会在Worker被调度的时候运行，而且，Worker必须被分配到一个逻辑CPU核心。如果你有四个核心5个Worker，其中一个Worker就不能工作。

I/O密集型的任务包含查询外部服务提供商，等待反馈。在一个执行I/O密集型任务的Worker在等待响应的时候它没有什么其他的事可以做，可以将CPU使用权交出去让别的Worker使用。因此，I/O密集型的任务即使在相关联的线程没有运行的时候也可以取得进展。

如果你只用一个Worker Pool,那么这个WorkerPool对CPU密集和I/O密集任务的不同特性就会影响你程序的性能和表现。

因为这个原因，你应该维护另一个独立的Computation Worker Pool。

分流：结论

对于简单的任务，比如遍历一个任意长的数组，分区也许是个好选择。如果你的计算更加复杂，分流就是更好的选择：此时传递和序列化对象的代价被使用多个CPU核心带来的性能提升抵消了。

然而，如果你的服务器严重以来复杂的计算，你应该考虑Node是否真的合适。Node 在I/O-bound 的场景下表现非凡，但是对于大量的计算，他并不是最好的选择。

如果你选择分流这个选项，查看不要阻塞Worker Pool 这一部分。


<a id="org3f84085"></a>

# 不要阻塞Worker Pool

Node 有一个由k个Worker 组成的Worker Pool。如果你使用分流的方法，可能还有一个Computation Worker Pool。不管怎样，假设k远小于要同时服务的客户端的数量。这个是和Node "一个线程，一堆客户端"的哲学一脉相承。

和上面讨论的一样，每个Worker完成他自己的任务之后再进行Worker Pool 队列还中的下一个。

每个任务完成的时间不同。需要最小化任务完成时间之间的差异，应该使用任务分区来实现。


<a id="org059b649"></a>

## 最小化任务完成时间差异

如果一个Worker的当前任务比其他的任务昂贵得多，那么他就没办法处理其他正在等待的任务。换句话说，每个相对比较长的任务会在它完成之前将WorkerPool的大小减少一。这是我们不想看到的，因为，在某种程度上，Worker Pool 中的Worker更多，throughput就越大。

为了避免这个问题，你应该最小化任务之间的完成时间差异。


<a id="org057fbff"></a>

## 任务分区

时间花费不同的任务会降低Worker Pool 的吞吐量。为了最小化任务间的时间花费差异，应该尽量将任务划分成时间花费相似的子任务。当每一个子任务完成的时候提交下一个，最后一个子任务完成的时候提醒提交者。


<a id="orgcd202f3"></a>

## 避免任务分区

如果你能分得清短任务和长任务，那你就可以为每种任务创建一个Worker Pool。将不同的任务放在不同的Worker Pool 中执行。

这种方法避免了任务分区带来的额外开销。


<a id="org8191d44"></a>

## Worker Pool :结论

不论你是使用Node Worker Pool 还是使用自己维护的单独的Worker Pool。你应该优化Pool 的任务吞吐量。
为了达到这个目的，使用任务分区方法来最小化任务之间的时间花费差异。


<a id="orgd769cfa"></a>

# NPM 包的风险

NPM 上的大部分包都是第三方开发者开发的，并且通常只保证尽力了(best-effort guarantees)。一个开发者使用npm包时应该注意两件事，第二件事经常被忽略。

1.  它实现了声称实现的API吗？
2.  它的API会阻塞事件循环或者WorkerPool吗？很多库不标明他们API的时间复杂度。


<a id="org49e271f"></a>

# 总结

Node 有两种线程: 一个事件循环和k个Worker。事件循环负责JS回调和非阻塞I/O，Worker执行对应的c++代码，包括I/O密集型和CPU密集型的任务。这两种线程都不会同时处理多过一个任务。如果一个回调或者一个任务花费了很多时间，执行它的线程可能被阻塞了。如果你的应用包含阻塞的回调或者阻塞的任务，那么就可能导致吞吐量低，甚至会导致拒绝服务。

想要写一个高吞吐量，免疫DoS攻击的web server，你必须要保证在恶意输入时，事件循环和WorkerPool 都不阻塞。
