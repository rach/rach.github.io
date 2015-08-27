---
title:  "Recursive Regular Expression"
date:   2015-08-20
description: It's not really well known that regex support recursive pattern, this post introduces this feature.      
---

I was discussing with a colleague about a simple problem that his company was asking during an interview: _"Given a string composed from opened and closed parentheses, detect if all the parentheses are closed"_

    ((())(())) -> Ok
    ()()  -> Ok
    ()()) -> Wrong
    ())( -> Wrong

You can solve this problem with counter starting from 0 and increment by 1 when you met `(` and decrement by 1 when you met `)`. The sum needs to stay positive or equal to zero, otherwise it's invalid string. A basic function in python to do this check of parentheses could look like this:

{% highlight python %}

def check(val):
    counter = 0
    for c in val:
        if c == '(':
            counter += 1
        elif c == ')':
            counter -= 1
        else:
            raise AttributeError('invalid character in the argument')
        if counter < 0:
            return False
    return counter == 0:

{% endhighlight %}

It's not the most elegant piece of python code but would it be possible to do the same with a regular expression? And the answer is YES!  
Now, it's not possible to do it with the built-in `re` package in Python because it doesn't support recursive pattern! 

To solve this problem with a regular expression in Python then, you need 
to install the [regex package](https://pypi.python.org/pypi/regex) which is more compatible with [PCRE](http://www.regular-expressions.info/pcre.html).

PCRE 4.0 and later introduced [regular expression recursion](http://www.regular-expressions.info/recurse.html), this allow to re-execute all or a part of the regular expression on the unmatched text. To use recursive regex, you use `(?R)` or `(?0)`.  
When the regex engine reaches (?R). This tells the engine to attempt the whole regex again at the present position in the string. If you want only to reapply a specific part of the regex then you use the grouping index: `(?1)`, `(?2)`

Using this, we can solve more complex problems with regex. Let's start by a more simple one and try to detect palindromes:

{% highlight python %}

>>> import regex
>>> regex.search(r"(\w)((?R)|(\w?))\1", "kayak") is None
True
>>> regex.search(r"(\w)((?R)|(\w?))\1", "random") is None
False

{% endhighlight %}

Let's analyse and decompose this regex:

-  `(\w)` match a single alphabetic character. eg: _'k'_ 
-  `(\w)\1` match 2 identical alphabetic characters. `\1` match the same value than `(\w)` matched. The number `1` represent the group position. eg: _'aa'_, _'bb'_
-  `(\w)(\w?)\1` match 2 or 3 alphabetic characters where the first and the last are equal. eg: _'kak'_, _'kk'_
-  `(\w)(((\w)\4)|(\w?))\1` match a 3 or 4 characters palindrome. eg: _'kaak'_ or _'kak'_

With `(\w)(((\w)\4)|(\w?))\1`, you can see that we are repeating the same logic to add be able to match a palindrome of 1 character longer than `(\w)(\w?)\1`. Ideally we would like a way to make a loop or define a recursive pattern. Perfect that what this post is about and you can express that with `((?R)|(\w?))` which apply all the regex at the current position or stop if there is 1 or 0 character left to process `(\w?)`.   

You can play with this regex via this [link](https://regex101.com/r/cQ1uC9/1)

Let's come back to our initial problem of parentheses. Using what we learn with the palindrome example, we can write a regex to solve it.   
The answer is  `^(\((?1)*\))(?1)*$`, let see this regex in action:

{% highlight python %}

>>> import regex
>>> regex.search(r"^(\((?1)*\))(?1)*$", "()()") is not None
True
>>> regex.search(r"^(\((?1)*\))(?1)*$", "(((()))())") is not None
True
>>> regex.search(r"^(\((?1)*\))(?1)*$", "()(") is not None
False
>>> regex.search(r"^(\((?1)*\))(?1)*$", "(((())())") is not None
False

{% endhighlight %}

Let's analyse and decompose this regex:

- `^` match a start of the string 
- `$` match the end of the string
- `(\(\))` match open and close parenthesises `()` 
- `(\((?R)?\))` match parentheses like `((()))`    
- `(\((?R)*\))` match parentheses like `(()()())`   
- `(\((?1)*\))(?1)*` match parentheses like `(()()())(())` where `?1` is `(\((?1)*\))` 
- `^(\((?1)*\))(?1)*$` we add `^` and `$` to consume the all the string  


You can play with this regex via this [link](https://regex101.com/r/jQ6yG0/1)

If you ask about performance then the python code that we written at the start is more performing:

{% highlight python %}

>>> import regex

>>> %timeit -n1000 check("(()())") 
1000 loops, best of 3: 1.81 µs per loop

>>> %timeit -n1000 regex.search(r"^(\((?1)*\))(?1)*$", "(()())")
1000 loops, best of 3: 13 µs per loop

>>> comp = regex.compile(r"^(\((?1)*\))(?1)*$")
>>> %timeit -n1000 comp.match("(()())")
1000 loops, best of 3: 7.49 µs per loop

{% endhighlight %}

The syntax above is base on `ipython` which allow to execute `timeit` with the syntactic sugar `%timeit`. 

This test was done on my laptop, but otherwise you can see that the simple python code 
is much faster than using a regex for this problem. 
It's not a surprising result because this problem can be solved in _O(n)_ and even if I don't know the complexity of applying a regular expression, I expect it to be bigger than O(n) to parse the input and doing some kind of recursivity. Still, I was curious try it because it's difficult to anticipate some behavior in Python when a component is written in C with Python.

I hope that this post will give a taste of advanced features possible with regular expressions.     

