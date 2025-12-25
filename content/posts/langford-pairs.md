+++
title = "Langford Pairs"
date = "2017-01-10"

[taxonomies]
tags=["java", "algorithms", "maths"]
+++

Recently I was asked to write some code to generate a [Langford Pair](https://en.wikipedia.org/wiki/Langford_pairing). I had never heard of those before.

A langford sequence is a sequence of numbers where every number appears twice and equal numbers have a difference of their positions equal to their value.

For example for n = 4 :       `4 1 3 1 2 4 3 2`

Clearly, the sequence is of length 2n as every number appears twice.

Intuitively we fix the highest number at the first position and at the next allowed position :   `4 _ _ _ _ 4 _ _`

Then we go to the next number, that is 3 and try to place it at the next possible position :   `4 _ 3 _ _ 4 3 _`

Now we try to place 2 : `4 2 3 _ 2 4 3 _`

Oops. There is no valid position left for 1 now. So we backtrack, and remove the last number inserted and re-insert that somewhere else. If no re-insertable position found for that number, we backtrack again to the next upper level and try re-position that number and so on.

So we backtrack, and remove 2 from its inserted position and check if we have any available position for 2. Turns out we have : `4 _ 3 _ 2 4 3 2`

Now, we found one available position for the 1s in the sequence, and we get the final Langford sequence after placing the 1s there :  `4 1 3 1 2 4 3 2`

Interesting to see that there are multiple Langford sequences for the same value of n. For example, if we reverse the previous Langford sequence, it will still be a Langford sequence. So for n = 4, we have 2 such sequences available.

So, how do we generate such a sequence ?

As I wrote above, it can be easily done using recursion. Define a function that takes an array, and the next number to insert. Now, if it finds a place to insert the number, it inserts them, and recursively calls itself to insert n-1 . If it can’t insert the number it returns, and the calling function will then try re-positioning n  and do the recursion again.

Turns out it is an expensive process, as we cannot use tail-recursion here. (For that matter in any backtracking algorithm).

But, will we always get a Langford sequence for any given number ? Here is a mathematical proof that it can only be feasible when  `n MOD 4 = 0` or `n MOD 4 = −1`

That saves us from doing useless recursions and overflowing the stack when we already know no such sequence exists.

We can also do it iteratively without recursion. But then, since we need to backtrack when no position is available for a number, we have to keep track of the numbers inserted using a stack.

The iteration can also be done in 2 different ways. Either we iterate on the positions, and see what number can be inserted at this position, or we iterate over the numbers, where we answer ‘at what position this number can be inserted’ in every loop. Both will result in different sequences.

We can generate all Langford Sequences by back tracking and re-positioning the numbers to the next available position.

[Here](https://github.com/gagan405/Algorithms/blob/master/algorithms/src/in/cafeaffe/algo/LangfordPairs.java) is the code that I wrote in some of my spare time, which implements all these 3 approaches.

References :

1. http://datagenetics.com/blog/october32014/index.html
2. http://dialectrix.com/langford/langford-algorithm.html