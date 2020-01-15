---
title: "Bash脚本总结"
layout: post
tags: bash
---

# Table of Contents

1.  [bash基础回顾](#org09d40d4)
    1.  [局部变量覆盖(shadowing)和dynamic scoping](#org669fa0a)
    2.  [使用 `exec` 重定向当前shell中命令的输出](#orgfaeff93)
    3.  [parameter expansion](#org88725dc)
    4.  [`echo` 和 `$()`](#org722b1bf)
2.  [有用的命令](#org74caf10)
    1.  [`read`](#orgf657a2c)
    2.  [`cut`](#orgca7be2c)
    3.  [`realpath`](#org12aba72)
    4.  [`set`](#orgd359853)

上周老大要我写个自动化的脚本，写这个脚本的过程中巩固了bash的一些知识，也学到了一些之前不懂的东西。写完这个脚本之后感觉很好，但是过了不到一周，就忘的差不多了。所以写一篇笔记总结一下。


<a id="org09d40d4"></a>

# bash基础回顾


<a id="org669fa0a"></a>

## 局部变量覆盖(shadowing)和dynamic scoping

这里不涉及 `subshell` 相关的问题，只讨论一个进程内，函数中的局部变量和函数外的全局变量。

bash不使用静态作用域(static scoping)，而是使用动态作用域(dynamic scoping)，这个特性还挺有意思。

dynamic scoping就是说，函数在 **调用栈** 里去找使用到的变量，而 static scoping 是指函数在它的 **定义** 位置去找使用到的变量。

下面代码中的 `testfun2` 就是一个dynamic scoping 的例子。

按照static scoping, `testfun2` 里面这个 `hello` 变量永远都是指向全局变量 `hello` ,在下面的代码中，这个全局变量的值一直都是 `1` 。

换句话说，如果使用static scoping， 不论在哪里调用 `testfun2` ,输出的都是全局变量 `hello` 的值，和局部变量没有任何的关系。

而下面代码的执行结果却不是这样： `testfun1` 中调用 `testfun2` 时，栈中存在两个 `hello` 变量，一个局部变量，一个全局变量，输出的是局部变量的值。

为什么没有输出全局变量的值呢？

因为bash使用dynamic scoping, 在用到 `hello` 这个变量时，它先在最近调用栈中找 `hello`, 找到的是局部变量。

这个特性在写一些脚本的时候可以提供很多方便，比如可以将一些参数作为调用处的局部变量，这样就可以少传几个参数。

{% highlight bash %}
hello="1"

testfun1() {
    local hello="world" # 局部变量
    echo $hello # 局部变量 hello 覆盖全局变量 hello
    hello="bye" # 修改的是局部变量，全局变量不受影响
    testfun2 # dynamic scoping, testfun2 会先从调用环境里面找 hello 这个变量，此处就是 testfun1 中的 hello
}

testfun2() {
    echo $hello
}

echo $hello
testfun1
echo $hello
testfun2 # 在这个调用中，testfun2 会获取到全局的 hello 变量
{% endhighlight %}

下面是输出：

| 1 |
| world |
| bye |
| 1 |
| 1 |


<a id="orgfaeff93"></a>

## 使用 `exec` 重定向当前shell中命令的输出

bash里面的重定向蛮复杂的，有好几种，这里做一个不完全的总结。

在写后端程序的时候，有一个习惯就是将程序的输出进行各种重定向，比如

1.  错误/异常输出到error.log，
2.  所有的消息都输出到all.log，
3.  此外还要输出一个进行了各种上色和格式化的版本到控制台。

其实写脚本也可以这么干。

{% highlight bash %}
exec 3>&1 1>>"${LOGFILE}" 2>>"${ERRLOGFILE}" 2>&1 2>&3
{% endhighlight %}

这条命令就可以达成1,2两个目标。标准错误(stderr)输出到 `$ERRLOGFILE` ,标准输出(stdout)和stderr都输出到 `$LOGFILE` 。

上面这条命令开头一个不带命令的 `exec` ，就是说它会影响当前shell里面的所有命令。 

`exec 3>&1` 这一段是打开 `/dev/fd/3` ，并且"复制(保存)"一份 `/dev/fd/1` 也就是标准输出(stdout)到 `/dev/fd/3` ，换句话说，原来输出到标准输出的，现在输出到标准输出和 `/dev/fd/3` ，原来链接到标准输出的，现在连接到标准输出和 `/dev/fd/3` 。

也就是说， `/dev/fd/3` 和标准输出是"一样"的了。

再换句话说， `/dev/fd/3` 现在指向 `/dev/fd/1` (可以理解成 `FILE* a = b` 这种)。

`1>>"${LOGFILE}" 2>>"${ERRLOGFILE}"` 这一段是比较基本的用法，将stdout附加到 `$LOGFILE` , stderr 附加到 `$ERRLOGFILE` 。 

`2>&1 2>&3` 这一段就是让2指向1, 2 指向3。有一个类似的指令交换 stdout 和stderr:

{% highlight bash %}
3>&1 1>&2 2>&3
{% endhighlight %}

还有一个挺有意思的问题是：这些重定向的顺序为什么是这样的？

实际上 `>&` 这个操作符使用了 `dup` 系统调用来实现复制file descriptor的效果。由于这个原因，需要先打开1,3然后将2重定向到1,3上面去。原理和 `1>$logfile 2>&1` 是一样的。


<a id="org88725dc"></a>

## parameter expansion

想要做一些简单的字符串匹配和替换操作时，这个特性非常好用，例如，简单的字符串替换

{% highlight bash %}
"${BRANCH/'refs/heads/'/''}" 
{% endhighlight %}

上面的例子中, `BRANCH` 是一个变量，存储着一个git分支的名字，但是形式是 `refs/heads/master` 这种，不好用，需要删掉 `refs/heads/` 这部分。

上面的例子中使用单引号将 `'refs/heads/'` 包起来，这样 `refs/heads/` 就是literal了，bash不会对他们做任何处理。

例子中 `${VARIABLE/pattern/replacement}` 这个格式的意思是，用 `replacement` 代替 `$VARIABLE` 这个变量中满足 `pattern` 的第一个地方。

类似的，还有很多操作符，可以替换所有的匹配，也可以做其他的事情。这个特性用起来很方便。在<https://guide.bash.academy/expansions/> 这个教程里面写的很详细。


<a id="org722b1bf"></a>

## `echo` 和 `$()`

在函数中，比如这个函数叫 `functionx` ，最后一句写 `echo xxx` ,那么， `echo "$(functionx)"` 的输出就是 `xxx` 。

这个特性可以用来传递一些值，“冒充”函数的返回值。因为实际上 `echo` 出来的东西并不是返回值，函数的返回值是一个数字，0-255， 0 表示成功，1-255表示出问题了。

这个值是函数中执行的最后一条命令的返回值，总之是一个数字。但是很多时候需要返回一些字符串，就可以使用 `echo` 和 `$()` 这个组合。 `$()` 叫做 command substitution, 旧的写法是 `` `` ``, 简单来说，它会获取括号(或者\`\`)里面命令的输出。


<a id="org74caf10"></a>

# 有用的命令


<a id="orgf657a2c"></a>

## `read`

这个命令是bash内建的，可以用来读取输入并存储到一个变量里面，在交互式的脚本里面可以做一些简单的问答。

比如:

{% highlight bash %}
read -p "请问你要买几斤橘子？" NUM
{% endhighlight %}

`-p` 的意思是 prompt，也就是打印一句话出来，然后读入变量。对于简单的交互，这个命令就足够处理了。


<a id="orgca7be2c"></a>

## `cut`

有时候需要处理一些有规律的文本，将一行分成几个部分然后取其中一部分。这时候cut甚至比sed，awk还好用。

比如，上面的例子中，需要将 `refs/heads/master` 变成 `master` 。

除了删除 `refs/heads/` ,还可以使用 `/` 作为分隔符，将字符串分割成 `refs heads master` 三个部分然后取第三个部分。使用 `cut` 可以很方便地办到这件事：

{% highlight bash %}
echo 'refs/heads/master' | cut -d/ -f3
{% endhighlight %}

`-d/` 的意思使用 `/` 作为分隔符， `-f3` 的意思是只留下第三部分。有意思的是， `cut` 是从一开始数的。
还有一个例子就是取得java版本号，使用 `cut` 可以缩短命令长度:

{% highlight bash %}
JAVAVERNUM=$(java -version |& grep 'version' | cut -d. -f2) # java -version 的输出在stderr......
{% endhighlight %}

上面是获得java版本号的命令，下面是 `java -version` 的输出。

    openjdk version "1.8.0_232"
    OpenJDK Runtime Environment (build 1.8.0_232-b09)
    OpenJDK 64-Bit Server VM (build 25.232-b09, mixed mode)

脚本运行之后, `JAVAVERNUM` 的值是 `8` (如果java版本是11 的话就是 `11` ，总之就是一个数字)。

使用 `|&` 这个管道的原因是 `java -version` 的输出在stderr， 直接 `|` 不行。

使用 `grep` 选出含有 `version` 的一行之后，应该只剩下第一行了，也就是 `openjdk version "1.8.0_232"`

其实第一个小数点后面的就是我们需要的版本好数字了。所以直接 `cut -d. -f2` 选出小数点分隔之后的数组中的第二个元素（从一开始数），可以不使用复杂的模式匹配就得到版本号。


<a id="org12aba72"></a>

## `realpath`

这个命令来自 `coreutils` 。可以把相对路径转换成绝对路径。这样一来，脚本里面就可以方便地全部使用绝对路径了，还不用自己拼接路径，不会出错。


<a id="orgd359853"></a>

## `set`

这个命令比较复杂，manpage里面的简介是 `set or unset options and positional paramenters` 。到目前为止我用到的就只有 `set -e` ,使脚本遇到命令出错(返回值不是0)就退出。
