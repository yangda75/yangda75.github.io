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
    `<`和`>`号在和`[[`一起使用的时候,会使用当前的locale，进行lexical排序。  
    在`test`命令的描述中查看参数的处理。  
    
```
       When  the  == and != operators are used, the string to the right of the operator is considered a pattern
       and matched according to the rules described below under Pattern  Matching,  as  if  the  extglob  shell
       option  were  enabled.  The = operator is equivalent to ==.  If the nocasematch shell option is enabled,
       the match is performed without regard to the case of alphabetic characters.  The return value  is  0  if
       the  string  matches  (==) or does not match (!=) the pattern, and 1 otherwise.  Any part of the pattern
       may be quoted to force the quoted portion to be matched as a string.

       An additional binary operator, =~, is available, with the same precedence as ==  and  !=.   When  it  is
       used,  the  string to the right of the operator is considered an extended regular expression and matched
       accordingly (as in regex(3)).  The return value is 0 if the string matches the pattern, and 1 otherwise.
       If  the  regular  expression is syntactically incorrect, the conditional expression's return value is 2.
       If the nocasematch shell option is enabled, the match is performed without regard to the case of  alpha‐
       betic  characters.  Any part of the pattern may be quoted to force the quoted portion to be matched as a
       string.  Bracket expressions in regular expressions must be  treated  carefully,  since  normal  quoting
       characters  lose their meanings between brackets.  If the pattern is stored in a shell variable, quoting
       the variable expansion forces the entire pattern to be matched  as  a  string.   Substrings  matched  by
       parenthesized subexpressions within the regular expression are saved in the array variable BASH_REMATCH.
       The element of BASH_REMATCH with index 0 is the portion  of  the  string  matching  the  entire  regular
       expression.   The  element  of  BASH_REMATCH  with index n is the portion of the string matching the nth
       parenthesized subexpression.

       Expressions may be combined using the following operators, listed in decreasing order of precedence:

              ( expression )
                     Returns the value of expression.  This may be used to override the  normal  precedence  of
                     operators.
              ! expression
                     True if expression is false.
              expression1 && expression2
                     True if both expression1 and expression2 are true.
              expression1 || expression2
                     True if either expression1 or expression2 is true.

              The  && and || operators do not evaluate expression2 if the value of expression1 is sufficient to
              determine the return value of the entire conditional expression.
```
当使用`==`和`!=`时，操作符右边的字符串被视为一个pattern，按照下面的`PatternMatching`章节中的规则来匹配，和`extglob`选项开启了一样。　`=`操作符和`==`等价。如果`nocasematch`选项开启了的话，匹配时会忽略字母的大小写。如果字符串匹配(`==`)pattern或者不匹配(`!=`)，返回值是0，否则是1。pattern中的任何一部分都可以被引号包围，强制按字符串匹配。  
另一个二元操作符`=~`，优先级和`==`和`!=`相同。当使用它时，右边的字符串被视为扩展正则表达式，并且按照规则匹配。如果字符串符合规则，返回值是0，否则是1。如果正则表达式的语法错误，那么条件表达式的返回值是2。如果`nocasematch`选项开启了，匹配时不考虑字母的大小写。pattern中的任意部分都可以被引号括起来，强行作为字符串匹配。要小心对待正则表达式中的括号表达式，因为常规的引号在括号中失去意义。如果pattern保存在一个shell变量中，用引号包围变量展开会强制将整个pattern作为字符串匹配。正则表达式中被括号子表达式匹配的子串被保存在数组变量`BASH_REMATCH`中。  
    `BASH_REMATCH`的`0`号元素是字符串匹配整个正则表达式的部分。`BASH_REMATCH`中下标为`n`的元素是字符串中匹配第`n`个括号内的子表达式的部分。  
    表达式可以用下面的操作符结合起来，按优先级递减排列：  
    `(expression)`返回`expression`的值。这可以用来覆盖操作符的优先级。  
    `! expression` 如果`expression`是错误的，为真。  
    `expression1 && expression2` 1,2都为真时为真。  
    `expression1 || expression2` 1,2至少有一个为真。  
    如果第一个表达式就可确定返回值，`&&` 和`||`操作符不会计算第二个表达式的值。
```
       for name [ [ in [ word ... ] ] ; ] do list ; done
              The list of words following in is expanded, generating a list of items.  The variable name is set
              to each element of this list in turn, and list is executed each time.  If the in word is omitted,
              the for command executes list once for each positional parameter  that  is  set  (see  PARAMETERS
              below).   The  return status is the exit status of the last command that executes.  If the expan‐
              sion of the items following in results in an empty list, no commands are executed, and the return
              status is 0.

       for (( expr1 ; expr2 ; expr3 )) ; do list ; done
              First,  the arithmetic expression expr1 is evaluated according to the rules described below under
              ARITHMETIC EVALUATION.  The arithmetic expression expr2 is then  evaluated  repeatedly  until  it
              evaluates  to  zero.   Each  time  expr2  evaluates to a non-zero value, list is executed and the
              arithmetic expression expr3 is evaluated.  If any expression is omitted,  it  behaves  as  if  it
              evaluates  to  1.   The  return value is the exit status of the last command in list that is exe‐
              cuted, or false if any of the expressions is invalid.

       select name [ in word ] ; do list ; done
              The list of words following in is expanded, generating a list of  items.   The  set  of  expanded
              words  is  printed  on the standard error, each preceded by a number.  If the in word is omitted,
              the positional parameters are printed (see PARAMETERS below).  The PS3 prompt is  then  displayed
              and  a  line read from the standard input.  If the line consists of a number corresponding to one
              of the displayed words, then the value of name is set to that word.  If the line  is  empty,  the
              words  and  prompt  are displayed again.  If EOF is read, the command completes.  Any other value
              read causes name to be set to null.  The line read is saved in the variable REPLY.  The  list  is
              executed  after  each  selection until a break command is executed.  The exit status of select is
              the exit status of the last command executed in list, or zero if no commands were executed.

       case word in [ [(] pattern [ | pattern ] ... ) list ;; ] ... esac
              A case command first expands word, and tries to match it against each pattern in turn, using  the
              same  matching  rules  as  for  pathname  expansion  (see Pathname Expansion below).  The word is
              expanded using tilde expansion, parameter and variable expansion, arithmetic  expansion,  command
              substitution,  process  substitution  and quote removal.  Each pattern examined is expanded using
              tilde expansion, parameter and variable expansion, arithmetic  expansion,  command  substitution,
              and  process  substitution.   If  the nocasematch shell option is enabled, the match is performed
              without regard to the case of alphabetic characters.  When a match is  found,  the  corresponding
              list  is  executed.   If  the  ;; operator is used, no subsequent matches are attempted after the
              first pattern match.  Using ;& in place of ;; causes execution to continue with the list  associ‐
              ated  with  the next set of patterns.  Using ;;& in place of ;; causes the shell to test the next
              pattern list in the statement, if any, and execute any associated list  on  a  successful  match.
              The exit status is zero if no pattern matches.  Otherwise, it is the exit status of the last com‐
              mand executed in list.

       if list; then list; [ elif list; then list; ] ... [ else list; ] fi
              The if list is executed.  If its exit status is zero, the then list is executed.  Otherwise, each
              elif  list  is  executed  in turn, and if its exit status is zero, the corresponding then list is
              executed and the command completes.  Otherwise, the else list is executed, if present.  The  exit
              status is the exit status of the last command executed, or zero if no condition tested true.

       while list-1; do list-2; done
       until list-1; do list-2; done
              The  while  command continuously executes the list list-2 as long as the last command in the list
              list-1 returns an exit status of zero.  The until command is  identical  to  the  while  command,
              except that the test is negated: list-2 is executed as long as the last command in list-1 returns
              a non-zero exit status.  The exit status of the while and until commands is the  exit  status  of
              the last command executed in list-2, or zero if none was executed.
```
- `for name [ [ in [ word ... ] ] ; ] do list ; done`  
`in`后面的列表被展开，生成一系列的单位。`name`被轮流设置成里面的元素，并且每次都会执行`list`。如果`in`被省略了，`for`循环对每个位置参数执行一次`list`。返回值是最后执行的命令的返回值。如果`in`后面的列表展开之后为空，那么不执行任何命令，返回0。  
- `for (( expr1 ; expr2 ; expr3 )) ; do list ; done`  
首先，`expr1`按规则执行，然后`expr2`重复执行直到值为0。每次执行`expr2`得到一个非零值，都会执行`list`和`expr3`。如果哪个表达式被省略了，视作它的值为1。返回值是`list`中最后一个执行的命令的返回值。如果有不合法的表达式，返回`false`。  
- `select name [ in word ] ; do list ; done`  
`in`后面的列表被展开，生成一系列单位。展开的一系列`word`被输出到标准错误中，每项前面都有一个数字。如果`in`被省略了，就把位置参数打印出来。然后显示`PS3`提示符，并且从标准输入读入一行。如果输入行中含有一个和显式的`word`对应的数字，那么`name`的值就被设置为那个`word`的值。如果输入行为空，就再来一次。如果读到了`EOF`，命令执行完毕。其他的读入值都会导致`name`被设为`null`。读入的行保存在`REPLY`变量中。直到遇到一个中断命令， 每选择一次就执行一次`list`。返回值是`list`中最后一个被执行的命令的返回值。如果没执行任何命令，返回0。  
- `case word in [ [(] pattern [ | pattern ] ... ) list ;; ] ... esac`  
`case`命令首先展开`word`，然后轮流试`pattern`，使用路径展开的匹配规则。`word`用`tilde/parameter&variable/arithmetic expansion, command substitution, process substitution`的规则展开。如果选了`nocasematch`，匹配无视大小写。如果找到了一个匹配，执行相应的`list`。如果有`;;`符号，找到了一个之后就不找了。用`;&`代替`;;`，找到一个之后会继续在下一个`pattern`对应的`list`里面找。`;;&`会导致shell测试下一个表达式里的`pattern list`如果有的话，那就在每次成功匹配的时候执行对应的`list`。如果没有匹配，返回值是0。如果有匹配，返回值是最后执行的命令的返回值。  
- `if list; then list; [ elif list; then list; ] ... [ else list; ] fi`
`if list`先执行，如果返回值是0，那么执行`list`。否则轮流执行`elif list`，如果返回值是0，对应的`then list`被执行，并且命令结束。否则，执行`else list`，如果存在的话。返回值是最后执行的命令的返回值，如果没有一个条件为真，就返回0。  
- `while list-1; do list-2; done  
  until list-1; do list-2; done`
  只要`list-1`中的最后一个命令返回值为0，`while`就会不停执行`list-2`。`util`和`while`一样，不同之处就是在`list-1`返回值不为零的时候不停执行`list-2`。这两个命令的返回值是`list-2`中最后执行的命令的返回值，如果没有执行，返回0。
