---
layout: post
title: "GNOME Tali and Random Numbers"
category: programming
---

I sometimes play [GNOME Tali][tali]. While playing it, I felt that the dice
rolls aren't completely random. For example, sometimes when I zeroed out the
"full-house" row, the rolls for next 2-3 rounds were full-houses.

So I had to go and look at the source code. It turned out that my feeling
wasn't anything more than an illusion. You can read about this phenomena more
[here][dailymail].

But I found out that the implementation ```RollDie()``` had a subtle bug.

<!-- more -->

Here is how it was implemented:

{% highlight c %}
int
RollDie (void)
{
  double r = (double) rand () / RAND_MAX;
  return (int) (r * 6.0) + 1;
}
{% endhighlight %}

What this code intends to do is to generate a random number in range
$$[0..RAND\_MAX)$$, then scale it to range $$[0..1)$$, and then map it to a
number in set $$\{1, ..., 6\}$$.

But the problem is ...

## RollDie() could generate 7

[Documentation][rand-man] of ```rand()``` says it generates a pseudo-random
number in the range 0 to RAND_MAX **inclusive**.

So, ```RollDie()``` can return 7, which is not expected by definition of this
function.

The easiest way to solve this is to use ```rand() % 6 + 1``` instead.

## Distribution is not uniform

When I saw this code, I asked myself why are we doing floating point arithmetic
instead of a simple ```rand() % 6 + 1```?

My guess was that it was attempting to make the distribution uniform. If we use
```rand() % 6 + 1```, since RAND_MAX is usually $$2^k$$ and not a multiple of 6,
then first few elements of $$\{1, ..., 6\}$$ will have slightly more chance than the
rest of the numbers in the set.

But this is also a problem for the floating point arithmetic implementation. This
is because it still tries to map RAND_MAX values into 6 values, and inevitably
some of the values will have slightly more chance than others.

To make the distribution uniform, we can find the largest ```6k < RAND_MAX```,
and try to generate a number in range $$[0..6k)$$, and then scale it down to
$$\{1, ..., 6\}$$:

{% highlight c %}
int
RollDie (void)
{
  int range_max = RAND_MAX - RAND_MAX % 6;
  int r = 0;
  while ((r = rand()) >= range_max);
  return r % 6 + 1;
}
{% endhighlight %}

This is a bit similar to [how python does randint][randint].

## Conclusion

I thought using ```rand() % 6 + 1``` was close enough to uniformity, although not
completely uniform, and making it completely uniform made it much more complex.
So I submitted [a patch][patch] which was checked in with the following code:

{% highlight c %}
int
RollDie (void)
{
  return rand () % 6 + 1;
}
{% endhighlight %}

[tali]: https://wiki.gnome.org/Apps/Tali
[dailymail]: http://www.dailymail.co.uk/home/moslive/article-1334712/Humans-concept-randomness-hard-understand.html
[rand-man]: http://man7.org/linux/man-pages/man3/rand.3.html
[randint]: https://github.com/python/cpython/blob/fbb8131675cce3fb137d2aa2af09de129e81ddb8/Lib/random.py#L231
[patch]: https://github.com/GNOME/tali/commit/e560c7d16517c8181404dcd4d9e6e752c30802cd
