---
layout: post
title: "Leetcode: Longest Palindromic Substring"
tags: leetcode, Manacher's algorithm, string process
---

## the problem itself

Input: string s;
Output: the longest *substring* in s which is panlindromic.

This problem is tagged as `medium` on leetcode. It's no difficult
to solve but it's associated with an interesting algorithm.

## an intuitive solution
After some thoughts, I came up with a solution finishes in
O(n^2) time and no extra space, where n is the length of
given string:

**expand from the *core***
Each palindromic string has a *core* at the central position, it
can be either one character or two. A palindrome is symmetric to
its *core*.

```
a 'dual core' palindrome:

"... a b c d d c b a ..."
           ^ ^
a 'single core' case:

"... x y z s z y x ..."
           ^
```

Loop the given string s; expand from `s[i]` to find the longest
panlindromic substring using `s[i]` as its single core; if
`s[i]==s[i+1]`, expand from the dual core `s[i]+s[i+1]`.

**time complexity**
Each expansion is O(n), because the worst case is expanding
from the core of given string `s` when `s` as a whole is
palindromic. And expansion happens inside a for loop, so the
worst case complexity is O(n^2).

Adding special characters to the string can make sure that
there is only one *core* case. But it is not a must-do step in
this solution.

c++ code:

{% highlight c++ linenos=table %}
string longestPalindrome(string s) {
    int size = s.size();
    if (size < 2)
        return s;
    string ans = "";
    bool same = true;
    for (int i = 0; i < size - 1; i++) {
        same = s[i] == s[i + 1];
        if (!same)
            break;
    }
    if (same)
        return s;
    for (int i = 0; i < size - 1; i++) {
        // the singular "core" case
        string current = string(1, s[i]);
        int pred = i - 1;
        int succ = i + 1;
        while (pred >= 0 and succ < size and s[pred] == s[succ]) {
            current = s[pred] + current + s[succ];
            pred -= 1;
            succ += 1;
        }
        ans = ans.size() < current.size() ? current : ans;
        if (s[i] == s[i + 1]) { // the dual "core" case
            current = "";
            current += s[i];
            current += s[i + 1];
            int pred = i - 1;
            int succ = i + 2;
            while (pred >= 0 and succ < size and s[pred] == s[succ]) {
                current = s[pred] + current + s[succ];
                pred -= 1;
                succ += 1;
            }
        }
        ans = ans.size() < current.size() ? current : ans;
    }
    return ans;
}
{% endhighlight %}

This code is accepted, but there is actually an O(n) way.

## the Manacher's algorithm
The Manacher algorithm is an O(n) algorithm for solving this
particular problem.
There are already some decent articles about the algorithm:
1. [最好的解释是用中文写的](
  https://www.felix021.com/blog/read.php?2040)
  
2. [a good explanation](
  http://articles.leetcode.com/longest-palindromic-substring-part-ii/)

### why is it O(n)?
In some degree, the Manacher algorithm is a refined version
of expanding from the center method mentioned before.
What makes it O(n) is that it gives the starting "radius"
before each expansion.

My c++ implementation without preProcess, plz do it
yourself :)
{% highlight c++ linenos=table %}
string longestPalindrome(string s) {
    string t = preProcess(s);
    unsigned long size = t.size();
    int p[size];
    int centerWithMaxReach = 0;
    int maxReach = 0;
    for (int i = 0; t[i] != '$'; i++) {
        int i_mirror = 2 * centerWithMaxReach - i;
        p[i] = maxReach > i ? min(maxReach - i, p[i_mirror]) : 0;
        // expand
        while (i - 1 - p[i] > 0 and i + 1 + p[i] < size and t[i + 1 + p[i]] == t[i - 1 - p[i]]) p[i]++;
        // update centerWithMaxReach and maxReach
        if (i + p[i] > maxReach) {
            centerWithMaxReach = i;
            maxReach = i + p[i];
        }
    }

    unsigned long maxLen = 0;
    int center = 0;
    for (int i = 0; i < size - 1; i++) {
        if (p[i] > maxLen) {
            maxLen = (unsigned long) p[i];
            center = i;
        }
    }
    return s.substr((unsigned long) ((center - 1 - maxLen) / 2), maxLen);
}
{% endhighlight %}

Line 11 and line 9 are crucial.
Actually, the while loop in line 11 always spend O(1) time
within `maxReach`, which means scanned part will not be
scanned again. This is guranteed:

1. Since there is a preProcess procedure, there is only the
single core case,

2. each 'core' `s[i]` we scanned has a known 'radius' `p[i]`,
`max(i+p[i])=centerWithMaxReach+p[centerWithMaxReach]` is
the furthest we go for now,

3. `s[j]` is going to be expanded, what if
`s[j]<max(i+p[i])`, which means `s[j]` has been scanned?

4. because `i>2*centerWithMaxReach-i` where
`2*centerWithMaxReach-i` is the mirror of `i`, we know
`p[i_mirror]`, since `i` and `i_mirror` are chars in the same
palindrome, they are closely related, what is their
relationship?

5. If the expansion of `p[i_mirror]` stopped before it goes
out of the palindrome centered at `centerWithMaxReach`,
the expansion of `i` will never go beyond `maxReach`.
If the expansion of `p[i_mirror]` goes out of the palindrome
centered at `centerWithMaxReach`, there is no need to expand
`i` because `p[i]` will be `maxReach - i`.
Hence, canned chars (those come before `maxReach`) does not
need to be scanned again.
