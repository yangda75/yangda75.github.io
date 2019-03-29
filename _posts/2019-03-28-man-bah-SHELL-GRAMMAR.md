---
layout: post
title: "man bash SHELL GRAMMAR"
tags: bash, mannual, translation, notes, shell grammar
---

## man bash 的中文翻译：SHELL GRAMMAR|阅读笔记

```
SHELL GRAMMAR
   Simple Commands
       A  simple  command is a sequence of optional variable assignments followed by blank-separated words and
       redirections, and terminated by a control operator.  The first word specifies the command  to  be  exe‐
       cuted, and is passed as argument zero.  The remaining words are passed as arguments to the invoked com‐
       mand.

       The return value of a simple command is its exit status, or 128+n if the command is terminated by  sig‐
       nal n.
```

简单命令：　　　　  
	一个简单命令是一系列可选的变量赋值之后跟着一系列空格分隔的词和重定向，以一个控制符号结尾。第一个单词指定要执行的命令，它作为参数0被传入。剩下的单词作为参数传入命令。  
	一个简单命令的返回值是一个退出状态，或者128+n，如果这个命令被信号n终止。  
	
```
   Pipelines
       A  pipeline  is  a  sequence of one or more commands separated by one of the control operators | or |&.
       The format for a pipeline is:

              [time [-p]] [ ! ] command [ [|⎪|&] command2 ... ]

       The standard output of command is connected via a pipe to the standard input of command2.  This connec‐
       tion  is  performed before any redirections specified by the command (see REDIRECTION below).  If |& is
       used, command's standard error, in addition to its standard output, is connected to command2's standard
       input through the pipe; it is shorthand for 2>&1 |.  This implicit redirection of the standard error to
       the standard output is performed after any redirections specified by the command.

       The return status of a pipeline is the exit status of the last command, unless the pipefail  option  is
       enabled.   If  pipefail  is  enabled, the pipeline's return status is the value of the last (rightmost)
       command to exit with a non-zero status, or zero if all commands exit  successfully.   If  the  reserved
       word  !  precedes a pipeline, the exit status of that pipeline is the logical negation of the exit sta‐
       tus as described above.  The shell waits for all commands in the pipeline to terminate before returning
       a value.

       If  the time reserved word precedes a pipeline, the elapsed as well as user and system time consumed by
       its execution are reported when the pipeline terminates.  The -p option changes the  output  format  to
       that  specified  by  POSIX.   When the shell is in posix mode, it does not recognize time as a reserved
       word if the next token begins with a `-'.  The TIMEFORMAT variable may be set to a format  string  that
       specifies how the timing information should be displayed; see the description of TIMEFORMAT under Shell
       Variables below.

       When the shell is in posix mode, time may be followed by a newline.  In this case, the  shell  displays
       the  total user and system time consumed by the shell and its children.  The TIMEFORMAT variable may be
       used to specify the format of the time information.

       Each command in a pipeline is executed as a separate process (i.e., in a subshell).
```

管道：  
	一个pipeline是一系列被 `|` 或者 `|&` 分开的命令。管道的格式如下：  
	```
	[time [-p]] [ ! ] command [ [|⎪|&] command2 ... ]
	```   
	`command` 的标准输出通过管道连接到`command2`的标准输入。这个连接发生在单个命令中的重定向之前。 `|&`是`2>&1 | `的简写，它把前一个命令的标准输出和标准错误都连接到后一个命令的标准输入。  
	一个管道的返回值是最后一个命令的exit status，如果打开了`pipefail`选项，那么返回值就是最右边的一个非零返回值，如果全部都成功了，就是0。在官道之前加上`!`符号，得到的返回值是正常返回值的逻辑反。shell会等管道中的所有命令结束之后返回。  
	如果`time`这个保留字出现在官道之前，那么管道结束时，会打印出使用的用户时间和CPU时间。`-p`选项将输出格式改为POSIX。当运行在POSIX  
	模式下时，如果`time`后面紧跟着一个`-`，shell不会把它当成一个保留字。`TIMEFORMAT`变量可以用来格式化时间信息的输出。  
	处在POSIX模式下时，`time`后面可以跟一个换行。此时，shell打印自己和子进程使用的总时间。同样，`TIMEFORMAT`可以用来设置时间输出格式。  
	管道中的每一个命令都在一个新的进程中进行。  

```
   Lists
       A list is a sequence of one or more pipelines separated by one of the operators ;, &, &&,  or  ||,  and
       optionally terminated by one of ;, &, or <newline>.

       Of  these list operators, && and || have equal precedence, followed by ; and &, which have equal prece‐
       dence.

       A sequence of one or more newlines may appear in a list instead of a semicolon to delimit commands.

       If a command is terminated by the control operator &, the shell executes the command in the  background
       in  a  subshell.   The shell does not wait for the command to finish, and the return status is 0.  Com‐
       mands separated by a ; are executed sequentially; the shell waits for  each  command  to  terminate  in
       turn.  The return status is the exit status of the last command executed.

       AND  and  OR lists are sequences of one or more pipelines separated by the && and || control operators,
       respectively.  AND and OR lists are executed with left associativity.  An AND list has the form

              command1 && command2

       command2 is executed if, and only if, command1 returns an exit status of zero.

       An OR list has the form

              command1 || command2

       command2 is executed if and only if command1 returns a non-zero exit status.  The return status of  AND
       and OR lists is the exit status of the last command executed in the list.
```
列表  
	一个列表就是一队管道，由`;`　`&`　`&&`或者`||`分开。有时候以`;`　`&`或者换行结束。`&&`和`||`有相同的优先级；`&`和`|`有相同的优先级。一个或者多个连续的空格可能会在列表中代替分号分隔命令。  
	如果一个命令以`&`结束，那么shell会在后台新建一个subshell来运行它。shell不会等它结束，并且返回值是0。以`;`分隔的命令逐个执行，返回值是最后一个命令的返回值。  
	`AND`和`OR`列表是用`&&`和`||`连接起来的列表，他们是左结合的。`AND`列表：  
	`command1 && command2` 当且仅当`command1`返回值为0时`command2`才会执行。   
	`OR`列表：`command1 || command2`　当且仅当`command1`的返回值不为零是`command2`才会执行。   
	`AND` `OR`列表的返回值是列表中执行的最后一个命令的返回值。  
	

```
   Compound Commands
       A  compound  command  is  one of the following.  In most cases a list in a command's description may be
       separated from the rest of the command by one or more newlines, and may be followed  by  a  newline  in
       place of a semicolon.

       (list) list  is executed in a subshell environment (see COMMAND EXECUTION ENVIRONMENT below).  Variable
              assignments and builtin commands that affect the shell's environment do  not  remain  in  effect
              after the command completes.  The return status is the exit status of list.

       { list; }
              list  is  simply executed in the current shell environment.  list must be terminated with a new‐
              line or semicolon.  This is known as a group command.  The return status is the exit  status  of
              list.   Note  that  unlike the metacharacters ( and ), { and } are reserved words and must occur
              where a reserved word is permitted to be recognized.  Since they do not cause a word break, they
              must be separated from list by whitespace or another shell metacharacter.

       ((expression))
              The  expression is evaluated according to the rules described below under ARITHMETIC EVALUATION.
              If the value of the expression is non-zero, the return status is 0; otherwise the return  status
              is 1.  This is exactly equivalent to let "expression".

       [[ expression ]]
              Return  a status of 0 or 1 depending on the evaluation of the conditional expression expression.
              Expressions are composed of the primaries described below under CONDITIONAL  EXPRESSIONS.   Word
              splitting  and  pathname  expansion  are not performed on the words between the [[ and ]]; tilde
              expansion, parameter and variable expansion, arithmetic expansion, command substitution, process
              substitution,  and  quote  removal  are  performed.   Conditional  operators  such as -f must be
              unquoted to be recognized as primaries.

              When used with [[, the < and > operators sort lexicographically using the current locale.

       See the description of the test builtin command (section SHELL BUILTIN COMMANDS below) for the handling
       of parameters (i.e.  missing parameters).

```

混合命令：  
	具有以下特征之一的是混合命令。绝大多数情况下，命令描述中的list可以被一个或几个换行（代替分号）和剩下的命令分隔开。  
	`(list)` list在subshell中执行。变量赋值和影响shell环境的內建命令在命令结束时停止作用。返回值就是list的返回值。  
	`{list;}` list在当前的shell环境中执行。list必须以一个换行或者分号结束。这种叫group command。返回值是list的返回值。和
	metacharacter 圆括号`()`不同，花括号`{}`是保留字，只能出现在保留字可以出现的地方。 由于花括号不会break一个word，所以一定要
	用空格或者其他的metacharacter和list分开。   
	`((expression))` expression根据下面的ARITHMETIC EVALUATION部分的描述来执行。如果expression的值不为零，返回值就是零；
	否则返回值就是1。这和`let expression`是完全一样的。  
	`[[ expression ]]`expression是一个条件表达式，根据它的值来返回0或者1。其中的表达式根据下面的CONDITIONAL EXPRESSION部分的
	描述来构建。word splitting, pathname expansion在`[[]]`中不进行。tilde expansion, parameter and variable expansion
	,arithmetic expansion, command/process substitution, quote removal照常进行。条件操作符，如`-f`一定要被unquote才能被
	识别为primary。
	
