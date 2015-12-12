---
layout: post
title: "Haskell Skyline"
category: programming
---

Recently I started learning Haskell by studying the [Intro to FP Programming]
[fp-course] course on Edx. Since then, I try to model different problems using
Haskell.

One of these problems is the Skyline problem, which goes like this:

> You are given a set of rectangular buildings in a city, and you should return
> the skyline view of the city. Input is a sequence of tuples $$(x_{left}, height,
> x_{right})$$, each describing a building. The output is a sequence of pairs
> $$(x, height)$$ meaning that the height of skyline changed to $$height$$ at
> the given x coordinate.

<!-- more -->

For example, for the input:

$$[(1,11,5), (2,6,7), (3,13,9), (12,7,16), (14,3,25), (19,18,22),\\
   (23,13,29), (24,4,28)]$$

The output should be:

$$[(1,11),(3,13),(9,0),(12,7),(16,3),(19,18),(22,3),(23,13),(29,0)]$$

Below diagrams show the input buildings and the expected skyline side by side:

   ![Skyline](/images/skyline.svg)


## Solving the problem

You can find an extended analysis of this problem at [Brian Gordon's blog][bgordon]
, which explains several solutions of $$O(n^2)$$ and $$O(n log n)$$ complexity
to this problem.

Since sorting of integers can easily be reduce to this problem in linear time,
this problem isn't solvable faster than $$O(n log n)$$ time.

In this post I'm not concerned with the time complexity at all, but mostly with
the simplicity of the solution.

It's also okay if a solution if contains a false height change, i.e. if we have
two consecutive items in the output that contain the same height. After all, we
are only concerned with how the skyline will look, and having a false change
doesn't change the shape of skyline.


### Solution 1. Calculate height of skyline at all endpoints

We're only interested at changes to the height of skyline, and height can only
change where a building starts or ends. The height at a point can be calculated
by finding the max height of buildings overlapping that point.

This leads us to the following solution:

{% highlight haskell %}
skyline bs = sort (foldl add_endpoints [] bs)
             where
                add_endpoints = 
                  \xs (x1, h, x2) -> xs ++ [(x1, height x1), (x2, height x2)]
                height =
                  \x -> maximum (0 : [h| (x1, h, x2) <- bs, x >= x1 && x < x2])
{% endhighlight %}

This solution has time complexity of $$O(n^2)$$ and produces some false changes,
but it is very simple and easy to understand.

(We can easily remove the false changes by passing the result through another
function, but this is good enough for our purpose and our goal is to keep things
simple).


### Solution 2. Iteratively add buildings

One might think we can iteratively add buildings and update the shape of skyline.
Since a building can be taller or shorter than the previously added buildings, we
may need to handle several cases to make this work correctly.

But if we sort the buildings by height before adding them, we can make the update
much simpler. If we know the height of skyline is shorter than current building,
then all we need to do is to remove the previous points between the boundries of
the current building (since they won't be visible anymore), and then add the two
points of building.

This leads us to the following solution:

{% highlight haskell %}
skyline bs = sort (foldl add_building [] (sortWith (\(_,h,_)->h) bs))
             where
                add_building = \xs (x1, h, x2) ->
                                  [(x, h)| (x, h) <- xs, x < x1 || x > x2] ++
                                  [(x1, h), (x2, 0)]
{% endhighlight %}

I think this also looks very simple, although not as simple as the previous
solution, with the advantage that it won't generate false height changes as
much as previous solution.

The time complexity of this solution is $$O(n^2)$$, but if we had used some sort
of sorted search tree with $$O(log n)$$ operations instead of using lists,
this could easily be improved to $$O(n log n)$$. After all what we do here is one
simple iteration with ```foldl```, and then adding each point exactly once, and
removing each end point at most once.


### Solution 3. Divide and Conquer

Since this is functional programming and solutions usually tend to be solved using
recursion, we could use a solution similar to merge sort. That is, solve the shape
of skyline for each half of the buildings, and then merge the shapes.

The solution will look like:

{% highlight haskell %}
skyline [] = []
skyline [(x1, h, x2)] = [(x1, h), (x2, 0)]
skyline bs = (skyline (take n bs), 0) `merge` (skyline (drop n bs), 0)
              where n = (length bs) `div` 2

merge ([], _) (ys, _) = ys
merge (xs, _) ([], _) = xs
merge ((x, xh):xs, xi) ((y, yh):ys, yi)
  | x > y = merge ((y, yh):ys, yi) ((x, xh):xs, xi)
  | x == y = (x, max xh yh) : merge (xs, xh) (ys, yh)
  | max xi yi /= max xh yi = (x, max xh yi) : merge (xs, xh) ((y, yh):ys, yi)
  | otherwise = merge (xs, xh) ((y, yh):ys, yi)
{% endhighlight %}

The ```merge``` function doesn't seem very simple, so I wouldn't use this solution
if I could use a simpler solution.

Similar to merge sort, the time complexity of this solution is $$O(n log n)$$.


### Conclusion

There are other solutions to this problem too, but I think these three solutions
are good representative of the solution space. I love problems which can be solved
in several totally different ways, and I found it beautiful that some of the
solutions are strikingly simple and easy to understand.


[fp-course]: https://www.edx.org/course/introduction-functional-programming-delftx-fp101x-0
[bgordon]: https://briangordon.github.io/2014/08/the-skyline-problem.html