---
layout: post
title: "python实现对称群运算"
mathjax: true
tags: python, basic group theory, recursion
---


## 解释

 $$  S  $$ 是一个有限集合，  $$\lvert S \rvert=n $$ 。所有从 $$ S $$ 到 $$ S $$ 的双射，和置换的乘法构成群。这个群成为n阶对称群，记为 $$ S_{n} $$ 。即 $$ S_{n} =(\{f \vert f:S\leftrightarrow S\}, \circ) $$ 下面用python实现这个群的运算，也就是置换的乘法。本篇文章中对称群的元素全部用轮换积表示。

### 例子
写这个程序的原因是为了写作业。有一道题目要求计算验证$$ K_{4} $$ = {(1),(12)(34),(13)(24),(14)(23)}是不是$$S_{4}$$的正规子群。最不用脑筋的想法就是拿$$K_{4}$$中的元素逐个左乘、右乘$$S_{4}$$，然后比较是不是相等。为了算 $$ k\circ S_4, \forall k \in K_4 $$ ，就有了这个程序。
举一个例子，来看看程序都要做什么。
计算(12)(34)(123)时，从最右边的一个轮换开始，取一个元素，找它的像。找像的过程就是映射的乘法： $$ (f \circ g)(a) = f(g(a)) $$ 。一个轮换就是一个映射。首先从最右边的括号里拿出2，最右边的轮换把2映射到3，3作为原像传给下一个映射，也就是从右数第二个括号，在第二个括号中，3被映射到4，4在第一个括号代表的映射中映射到4。这样得到2最终被映射到4。重复以上的步骤，直到所有的元素都被考虑过。得到结果(243)。

### 抽象
为了让程序代替我计算这些乘法，我得一步一步告诉它怎么做。

#### find_next(expression,a)
从上面的例子中可以得出：“寻找一个元素的像”这个过程进行了很多次，是运算中的基础部件，我们先来考虑这个过程。寻找一个元素的像，把这个过程用find_next函数表示。find_next函数接收两个输入，一个是要计算的算式，另一个是要找到像的元素。回忆上面的例子，find_next要做的其实就是我们得到３之后做的事情，(find_next((12)(34),3))。首先是找到式子中３第一次出现的位置，从右往左找，第一次出现之前每一个括号对３来说都是恒等映射，不用管。找到之后就进行find_next((12),4)，因为3被映射到４了，剩下的式子就是(12)。到这里就发现了find_next可以写
成递归形式的，其实也可以写成while循环，不过这里递归容易想到。
最终的代码：
{% highlight python %}
{% raw %}
def find_next(expression, a):
    l = len(expression) - 1
    while l >= 0:
        if a in expression[l]:
            break
        else:
            l -= 1
    if l > 0:
        pos = 0
        for i in range(len(expression[l])):
            if expression[l][i] == a:
                pos = i
                break
        return find_next(expression[:l:],
                         expression[l][(pos + 1) % len(expression[l])])
    else:
        if a in expression[0]:
            return expression[0][(
                expression[0].index(a) + 1) % len(expression[0])]
        else:
            return a
{% endraw %}
{% endhighlight %}


#### calc(expression)
回去看一下例子，对每个元素执行一遍find_next(expression,a)，就计算完了，而且两个元素的find_next过程还互不影响，令人开心。
所以，有了find_next函数就可以写计算乘法的主要过程了。再回去看一下例子，可以发现calc函数其实也是一个递归或者while循环：找出所有没计算过的元素，组成集合A，计算过的元素记为Ｂ，每次从Ａ中删掉元素ａ，执行find_next(expression,a)，然后把a放在Ｂ中。结果有三种可能：    
	1. a: calc(expression)的结果把a映射到a，这种情况下a不用写在轮换积表示中，随便找个A中的元素执行find_next就行    
	2. Ａ中一个元素k，执行find_next(expression,k)    
	3. B中一个元素g，结合轮换的知识，这种情况表示轮换结束了，随便找一个A中元素执行find_next    
在这一步中我觉得while循环比较自然，所以就写成了while循环。
最终的代码：
{% highlight python %}
{% raw %}
def calc(expression):
    ans = []
    partial_ans = []

    flatten = lambda l: [item for sublist in l for item in sublist]
    untouched_nums = set(flatten(expression))
    while (len(untouched_nums) != 0):
        if partial_ans == []:
            partial_ans.append(untouched_nums.pop())
        else:
            next_num = find_next(expression, partial_ans[-1])
            while next_num not in partial_ans:
                partial_ans.append(next_num)
                untouched_nums.remove(next_num)
                next_num = find_next(expression, partial_ans[-1])
            if len(partial_ans) != 1:
                ans.append(partial_ans)
            partial_ans = []
    return ans
{% endraw %}
{% endhighlight %}

#### 代码
在上面的实现中，恒等映射是[]。用来从表达式中取出所有元素的`flatten`这个lambda表达式写法是在StackOverflow上看到的。[stackoverflow](https://stackoverflow.com/questions/952914/making-a-flat-list-out-of-list-of-lists-in-python)    



> Written with [StackEdit](https://stackedit.io/)
