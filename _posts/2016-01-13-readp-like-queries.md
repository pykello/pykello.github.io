---
layout: post
title: "Matching LIKE Queries using Haskell, Part 1"
category: haskell
---

To practice more Haskell, I tried to solve problem [1177. Like Comparisons]
[problem-1177] from the Timus Online Judge.

The problem is to get some queries of form:

{% highlight sql %}
'string' like 'template'
{% endhighlight %}

and print "YES" if the input string matches the template, and NO otherwise.
Refer to the problem statement for the rules of matching and some examples.

The GHC compiler installed at Timus only contains the basic libraries, so
I couldn't use more popular parser libraries like [Parsec][parsec] or
[Attoparsec][attoparsec]. So I used the [Text.ParserCombinators.ReadP]
[readp] library.

In this post and up-follow posts I'll try to explain how I solved this probem
using ReadP. In doing so, I'll try to explain most of the details that aren't
clear for someone who is new to Haskell.

You can find my complete solution for this problem [here][gist].

<!-- more -->

## Representing a LIKE pattern as a type

Let's represent a LIKE pattern as the following type:

{% highlight haskell %}
data LikePattern = Nil | Wildcard LikePattern | InSet [Char] LikePattern
                       | NotInSet [Char] LikePattern
    deriving (Show, Eq)
{% endhighlight %}

This definition defines a new type named ```LikePattern```, which can be
constructed using the given constructors. Later, we will parse the input
pattern strings into this type and will use it to check if the given string
matches the pattern or not.

Here are some examples to help you understand this type better:

* Empty pattern translates to ```Nil```.
* ```"%a"``` (i.e. pattern matching strings ending with a) translates to
  {% highlight haskell %}
   Wildcard (InSet ['a'] (Nil))
  {% endhighlight %}
* ```"[abc]_[^xy]%"``` translates to
  {% highlight haskell %}
   InSet ['a','b','c'] (NotInSet []
                    (NotInSet ['x','y']
                    (Wildcard 
                    (Nil))))
  {% endhighlight %}

Note that we translate single character x to ```"InSet [x] ..."```, and
we translate single character wildcard ```_``` to ```"NotInSet [] ..."```.

Last bit of the above definition is:

{% highlight haskell %}
    deriving (Show, Eq)
{% endhighlight %}

This tells the Haskell compiler to automatically generate the ```show```
function (which converts a value to string), and equality operator for
our type. We could've implemented these operators and functions ourselves,
but since in this case these functions should be defined exactly based on
structure of our type, we let Haskell to auto-generate these functions for
us.


## Matching LIKE patterns with input strings

The code for this part is:

{% highlight haskell %}
matches :: String -> LikePattern -> Bool
matches "" p = (p == Nil) || (p == Wildcard Nil)
matches (t:ts) p = case p of
                    Nil             -> False
                    Wildcard rest   -> (matches (t:ts) rest) || (matches ts p)
                    InSet s rest    -> (elem t s) && (matches ts rest)
                    NotInSet s rest -> (not (elem t s)) && (matches ts rest)
{% endhighlight %}

First line tells that ```matches``` is a function which gets a string and a like
pattern, and returns a boolean value. This boolean value tells us if the
given string matches the given pattern or not.

Second line defines the result of the function if the given string is empty. If
our string is empty, then it matches only two patterns. Namely, ```"Nil"``` and
```"Wildcard Nil"```. Our code for translating to LikePatterns won't generate
consecutive Wildcard patterns, otherwise we should have also considered patterns
like ```"Wildcard (Wildcard (... (Wildcard Nil))"```.

Finally, the definition for non-empty strings.

The left handside of the definition, ```"matches (t:ts) p"```, names the first
character of the string ```"t"```, and the rest of string ```"ts"```, and the
pattern ```"p"```.

The right handside is a pattern matching based on the given pattern.

* If pattern is Nil, since our string is non-empty, the result is false.
* If pattern starts with the wildcard character, we have two cases.
  ```"matches (t:ts) rest"``` covers the case where the wildcard character
  consumes zero characters of the string, and ```"matches ts p"``` covers
  the case where the wildcard character consumers at least one character.
  If either of these cases returns true, we return true.
* ```"InSet s rest"```, then first character must be a member of the set
  ```"s"```, and ```"ts"``` should match rest of the pattern.
* ```"NotInSet s rest"```, is similar to the previous case.

In upcoming posts, I'll explain the parsing code.

[problem-1177]: http://acm.timus.ru/problem.aspx?space=1&num=1177
[parsec]: https://hackage.haskell.org/package/parsec
[attoparsec]: https://hackage.haskell.org/package/attoparsec
[readp]: https://hackage.haskell.org/package/base-4.8.1.0/docs/Text-ParserCombinators-ReadP.html
[gist]: https://gist.github.com/pykello/536dcde060db18c892d5