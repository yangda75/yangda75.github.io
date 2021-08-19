---
layout: post
title: "记一次debug"
tags: debug, wsl1, socket leak
---

## 我们至今仍未知道第一次出现bug的时间
```
8月7号，周六，中午12:57，
去现场实施的项目经理
在群里
说
“XXX系统又登不上去了，每天早上都会有一次”
```

当时他没有用`@`功能，当天也是周六，都在休息，没有人意识到这将是一次怎样的debug。
而小杨我，根本不记得看到过这条消息。

## 看看log
```
8月9号，周一，上午10:28，
去现场实施的项目工程师
在群里
说
10:28  > “@A @小杨 麻烦看一下log，8月7号晚上7点多左右xxx服务暂停了，以至于登录不上，查一下是什么原因。”
10:29  > 文件<xxx.log.rar>(微信电脑版)
```

这位同事使用了 `@` 功能，也发了log。而且是周一早上。虽然我刚到公司，还没完全睡醒，
还是很快地从一堆log里找到了“原因”。非常经典的一个报错：

    java.net.SocketException: No buffer space available (maximum connections
    reached?) 
google上可以搜到182,000,000条结果。最早的一条是2003年的(tomcat)[https://developer.jboss.org/thread/55838]
很开心，直接截图发到了群里。由于这一条报错是在程序刚启动，初始化数据库连接池的时候
报的，就顺便加了一句话，

    10:29 >“@A 是sqlserver连不上了吗？” 

现在再看，这是句废话，我真没睡醒。

一位没有被`@`的前端同事说是连接数不够了，没有释放。老大说是sqlserver（客户管理）
的连接数不够了。8月9号，问题第一次被处理，由于小杨多说的一句废话，新来的技术主管
A，老大（老王），还有比我们都更接近问题真相的前端同事，把这个宝贵的早上浪费在了
检查sqlserver连接数上，浪费了黄金时间。

## 干扰
1. 新系统bug多很正常，我们在这个项目中，用了三个新系统，(xxx1, xxx2, xxx3)，通过
http通讯。bug多点很正常
2. 三个系统中，一个可以跑在几乎所有主流平台上（flutter），
另一个也差不多（java），第三个只能跑在linux上
3. 客户和我们形成了鲜明的对比。他们只给了我们一个选择：windows server 2019,
build 17763(伏笔1)。由于我们三个月前没有通知要用linux，现在已经来不及申请了，先用
WSL(伏笔2)顶上（Vmware明显是更好的选择，但是客户给我们的虚拟机资源不够）
4. 频繁的需求变动，或者说，开发完成后，进场测试时才开始进行的“需求评审”

新系统的bug，以及大家都不太熟的WSL(不能用Linux真机，还有Vmware，谁用这种残废)，跨国公司の流程，废纸(还好我们没有把需求文档打印出来)一样的文档。

我迷失了。

我的能力是弱，但是“你跺你也麻”！

## `netstat`

    8月11日，周三，上午9:18,
    新来的技术主管A
    在群里
    说
    9:18  > <stackoverflow 链接1，java-netty-tcp-and-udp-connection-integration-no-buffer-space-available-for>
    9:18  > @老王 @小杨
    9:23  > <stackverflow 链接2，java-net-socketexception-no-buffer-space-available-maximum-connections-reached>
    
`netstat -ano`出场。但是出师不利，结果只有不到10k，windows默认分配一万多个动态端
口，出问题时netstat的输出大小单位应该是M。

该做的还是要做，确认了java的HTTP库有连接池，改了一个无关紧要的小bug，问题还是存在。


## 相信Windows

    8月12日，周四，下午13:20,
    小杨
    在群里
    说
    13:20  > 事件查看器
    13:20  > windows日志 > 系统
    13:20  > 来源为Tcpip的，看看有没有图里这种事件
    13:20  > 一张截图，分配端口失败的Windows事件

还好Windows记了这个日志。配合我们程序记的日志，终于确定了原因：系统的动态端口被
耗尽了。

## 脚本

    8月12日，周四，下午17:42，
    小杨
    在群里
    发了一个每隔两分钟执行一次 netstat -ano，将结果保存到指定文件夹的脚本
    
我们需要确认netstat的输出一直不到10k

## 你不可以永远相信Windows，破案

    8月18日，周三，上午9:57，
    去现场实施的项目经理
    在群里
    说
    9:57  > <Bound.txt, 9.3M> Get-NetTCPConnection的输出
    9:57  > 目前客户的反馈是，端口虽然被释放了，但处于bound状态，实际这个端口仍然是不可用的
    9:57  > 可能还需要优化下端口释放
    
这个文件大小对劲！我处理了Bound.txt，发现里面有三万多行的bound状态的socket记录，
都属于一个PID，9724。之后找到了一个stackoverflow的帖子，
[superuser](https://superuser.com/questions/1348102/windows-10-ephemeral-port-exhaustion-but-netstat-says-otherwise)
和我们遇到的问题相同：端口耗尽但是netstat输出很少，有一个程序跑在wsl里。帖子指向
一个 [GitHub
Issue](https://github.com/Microsoft/WSL/issues/2913#issuecomment-455262160)，里
面有复现方法，和修复说明：升级windows到build 18890。去现场实施的项目工程师同事复
现了(参见伏笔1,2)，公司的测试机和我们的开发机都没复现，因为我们的windows都比较新，
小杨作为全组唯一一个用wsl而不是Vmware或者linux真机开发的人，用的是wsl2(伏笔2)。

## 破案之后
bug找出来了，但是还没修复，因为 跨国公司の流程 。系统不可以随便升级。
这又是一个windows系统的bug，不升级解决不了。不过只是时间问题了。

    BUG(8月?日 - 8月18日),8月7日第一次被报告，8月18日查明来历

    完
