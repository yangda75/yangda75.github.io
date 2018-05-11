---
layout: post
title: "man bash INVOCATION to SHELL GRAMMAR"
tags: bash, mannual, translation, notes, shell grammar
---

## man bash 的中文翻译：(INVOCATION - SHELL GRAMMAR之前)|阅读笔记

```bash
INVOCATION
       A login shell is one whose first character of argument zero is a -, or one
       started with the --login option.

       An interactive shell is one started without non-option arguments and without 
       the -c option whose standard input and  error  are  both  connected  to  
       terminals  (as  determined  by isatty(3)), or one started with the -i option.
       PS1 is set and $- includes i if bash is interactive, allowing a shell script 
       or a startup file to test this state.

       The  following  paragraphs  describe how bash executes its startup files.  If
       any of the files exist but cannot be read, bash reports an error.  Tildes are
       expanded in filenames as described below under Tilde Expansion in the
       EXPANSION section.

       When bash is invoked as an interactive login shell, or as a non-interactive
       shell with the --login option, it first reads and executes commands from the
       file /etc/profile, if  that file  exists.   After  reading that file, it
       looks for ~/.bash_profile, ~/.bash_login, and ~/.profile, in that order, and
       reads and executes commands from the first one that exists and is readable.
       The --noprofile option may be used when the shell is started to inhibit this
       behavior.

       When a login shell exits, bash reads and executes commands from the file
       ~/.bash_logout, if it exists.

       When an interactive shell that is not a login shell is started, bash reads and
       executes commands from /etc/bash.bashrc and ~/.bashrc, if these files exist.
       This may  be  inhibited by using the --norc option.  The --rcfile file option
       will force bash to read and execute commands from file instead of /etc/bash.bashrc
       and ~/.bashrc.

       When  bash  is started non-interactively, to run a shell script, for example,
       it looks for the variable BASH_ENV in the environment, expands its value if it
       appears there, and uses the expanded value as the name of a file to read and execute.
       Bash behaves as if the following command were executed:
              if [ -n "$BASH_ENV" ]; then . "$BASH_ENV"; fi
       but the value of the PATH variable is not used to search for the filename.

       If bash is invoked with the name sh, it tries to mimic the startup behavior of
       historical versions of sh as closely as possible, while conforming to the  POSIX
       standard  as  well.
       When  invoked as an interactive login shell, or a non-interactive shell with the
       --login option, it first attempts to read and execute commands from /etc/profile
       and ~/.profile, in that order.  The --noprofile option may be used to inhibit this
       behavior.  When invoked as an interactive shell with the name sh, bash looks for
       the variable ENV, expands its value if  it  is defined, and uses the expanded value
       as the name of a file to read and execute.  Since a shell invoked as sh does not
       attempt to read and execute commands from any other startup files, the --rcfile
       option has no effect.  A non-interactive shell invoked with the name sh does not
       attempt to read any other startup files.   When  invoked  as  sh,  bash
       enters posix mode after the startup files are read.

       When bash is started in posix mode, as with the --posix command line option, it
       follows the POSIX standard for startup files.  In this mode, interactive shells
       expand the ENV variable and commands are read and executed from the file whose
       name is the expanded value.  No other startup files are read.

       Bash attempts to determine when it is being run with its standard input connected
       to a network connection, as when executed by the remote shell daemon, usually rshd,
       or the  secure shell daemon sshd.  If bash determines it is being run in this fashion,
       it reads and executes commands from ~/.bashrc and ~/.bashrc, if these files exist
       and are readable.  It will not do this if invoked as sh.  The --norc option may be
       used to inhibit this behavior, and the --rcfile option may be used to force another
       file to be read, but  neither  rshd  nor sshd generally invoke the shell with those
       options or allow them to be specified.

       If  the shell is started with the effective user (group) id not equal to the real
       user (group) id, and the -p option is not supplied, no startup files are read,
       shell functions are not inherited from the environment, the SHELLOPTS, BASHOPTS,
       CDPATH, and GLOBIGNORE variables, if they appear in the environment, are ignored,
       and the effective user id is  set  to the real user id.  If the -p option is supplied
       at invocation, the startup behavior is the same, but the effective user id is not reset.
```


登陆shell的零号参数的第一个字符是-,或者有 `--login` 选项。
交互式shell没有非选项参数也没有-c选项，而且它的标准输入和错误都和terminal链接，
(由 `isatty`确定)，或者有一个-i选项。如果shell是交互式的，PS1被设定而且$-的值中
有i,这使得脚本或者启动文件可以检查这个状态。

下一个段落描述bash是如何执行启动文件的。如果有文件存在但是无法被读取，bash会报
一个错。~被展开成文件名，展开方式在EXPANSION部分描述。

当bash以交互的登陆shell模式启动后，或者以一个`--login`选项启动后，它首先执行`/etc/profile`
中的命令，如果那个文件存在的话。在读取了那个文件后，它会寻找`~/.bash_profile`,`~/.bash_login`
和`~/.profile`，以上述顺序读取并执行第一个可读取文件中的命令。用`--nonprofile`选项来阻止这个行为。

当一个登陆shell存在时，bash读取并执行`~/.bash_logout`中的命令，如果它存在的话。

当一个不是登陆shell但是是交互的的shell启动时，bash读取并执行`/etc/bash.rc`和
`~/.bashrc`中的命令，如果这些文件存在的话。这行为可以被`--norc`选项阻止。`--rcfile`选项会强制bash执行一个文件中的命令而不是之前提到的两个文件中的。

当bash以非交互方式启动时，为了执行一个脚本，它会查看BASH_ENV中的变量，如果它
出现在其中就展开那个变量，并且把展开得到的值作为要读取并执行的文件名。bash就
像以下命令一般行为：
`if [-n "$BASH_ENV"]; then . "$BASH_ENV"; fi`
但是PATH变量并不会被用来搜寻文件名。

如果bash以sh的名字被启动，它尝试去模仿历史悠久的sh的启动行为，同时遵循POSIX
标准。当以交互式登陆shell或带`--login`选项的非交互式shell启动时，它首先读取并
执行`/etc/profile`和`~/.profile`中的命令，以上述顺序。`--noprofile`选项可以用来阻
止这个行为。当以sh的名字，交互模式启动时，bash会寻找ENV变量，如果有定义就展
开它的值，并且读取并执行这个名字的文件中的命令。因为以sh命令启动的shells不会
读取执行其他文件中的命令，`--rcfile`选项没有作用。一个以sh命令启动的非交互式
shell不会去读取任何其他的启动文件。当以sh命令被启动时，bash在读取完启动文件后
进入POSIX模式。

当bash以POSIX模式启动时，和有`--posix`选项时相同，它会遵从POSIX标准关于启动文件
的约定。在这种模式下交互式shell展开ENV变量并且读取并执行名字在展开值中的文件
中的命令。不会读取其他的启动文件。

bash尝试确定他的标准输入是否连接了网络，就像以远程守护程序（通常是rshd）或者
安全shell守护程序(sshd)启动时那样。如果bash确定它是如此运行的，它会从`~/.bashrc`
中读取并执行命令，如果这些文件存在并且可以读取的话。如果以sh启动，它不会这么做。
`--norc`选项可以阻止这种行为，`--rcfile`这个选项可以强制指定另一个文件来读取，但是rshd或sshd都没有这些参数也不支持指定这些选项。

如果启动shell的euid不等于ruid,而且没有-p选项，那么不会读取任何启动文件，
shell函数不是从环境中继承的，如果环境中存在SHELLOPTS,BASHOPTS,CDPATH和
GLOBIGNORE变量，它们会被无视。而且euid会被设置成ruid。如果有-p选项，那么启动
行为一样，但是euid不会被重设。


```
DEFINITIONS
       The following definitions are used throughout the rest of this document.
       blank  A space or tab.
       word   A sequence of characters considered as a single unit by the shell.
              Also known as a token.
       name   A word consisting only of alphanumeric characters and underscores,
              and beginning with an alphabetic character or an underscore.
              Also referred to as an identifier.
       metacharacter
              A character that, when unquoted, separates words.  One of the following:
              |  & ; ( ) < > space tab
       control operator
              A token that performs a control function.  It is one of the following symbols:
              || & && ; ;; ( ) | |& <newline>

```


定义：    
下列定义在整个文档中被使用。
blank 一个空格或者制表符
word  被当作一个单位处理的一系列字符，也称token
name  一个只由数字字母和下划线组成的，以字母或者下划线开头的word，也叫identifier
metacharacter 一个在不被引号包裹时分开word的字符，是下列符号之一：    
`      | & ; ( ) < > space tab `
control operator 执行一个控制功能的一个token，是下列符号之一：    
`      || & && ; ;; ( ) | |& <newline> `

```

RESERVED WORDS
       Reserved  words are words that have a special meaning to the shell.
       The following words are recognized as reserved when unquoted and
       either the first word of a simple command (see SHELL GRAMMAR below)
       or the third word of a case or for command:

       ! case  coproc  do done elif else esac fi for function if in select
       then until while { } time [[ ]]
```

保留字：
保留字是对shell由特殊意义的word。下面的word在没有被引用并且是简单命令中的第
一个字或是一个case或者命令的第三个字时是保留字：
``` ! case coproc do done elif else esac fi for function if in select then
until while { } time [[ ]] ```

