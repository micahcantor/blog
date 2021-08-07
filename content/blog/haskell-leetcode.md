+++
title = "Solving a few Leetcode questions in Haskell"
date = 2021-08-07
description = "Exploring some purely functional programming solutions to a few popular Leetcode questions."
[taxonomies]
tags = ["haskell", "leetcode"]
[extra]
katex = true
+++

[Leetcode](https://leetcode.com) is a popular site which hosts a database of programming questions and crowd-sourced solutions in different programming lanuages. Recently, I decided to try my hand at solving some of these questions using Haskell, both to improve my skills in the language, and to prepare for future interviews, where I may or may not use Haskell. Unfortunately, Leetcode does not support using Haskell to submit answers, but I thought it would be fun nonetheless.

One of the difficulties with solving Leetcode questions with a functional langauge like Haskell is that most, if not all, solutions and explanations for a given question are presented in an imperative or object-oriented style. Despite this, I think that thinking functionally can often lead to incredibly elegant solutions to these problems, so I'd like to share some of these interesting ones here. I've also tried to ensure that the solutions are algorithmically efficent -- no brute force answers allowed. Let's jump in.

## [Add Two Numbers](https://leetcode.com/problems/add-two-numbers/)

*You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse order, and each of their nodes contains a single digit. Add the two numbers and return the sum as a linked list.*

In this question, we are given two lists of digits in reverse order and we need to return the list of digits of their sum. For example:

```hs
add [2, 4, 3] [5, 6, 4] -- 465 + 243
  = [7, 0, 8] -- 807
```

Of course, the built-in list type in Haskell is a linked-list, which makes this questions particularly easy to handle. 

Our strategy here will simply be to convert both lists to their decimal representation, then add them and convert the resulting sum back to a list of digits. This will take $O(n)$ time, but might not be particularly efficient due to the conversion back and forth. However, it means that we don't have to worry about carrying digits, or what to do when the lists are of different lengths.

First we need a `toDecimal` function, to convert from a digit list to an integer, which we define like this, using point-free style:

```hs
-- ex: [2, 4, 3] == 342 == 2*10^0 + 4*10^1 + 3*10^2
powersOfTen = [10 ^ n | n <- [0 ..]]
toDecimal = sum . zipWith (*) powersOfTen
```

Here we define a lazy, infinite list of all powers of 10. Then to convert to decimal, we multiply each digit with its corresponding power, then sum it up.

Next we need a `fromDecimal` function which will essentially act in reverse: given a number, produce its digits in a list in reverse order. Notice that we want to generate a list from a single value, which is exactly what the `unfoldr` function from `Data.List` is meant to do! [^1]

Recall that `unfoldr` is a higher-order function with the type

```hs
unfoldr :: (b -> Maybe (a, b)) -> b -> [a]
```

Essentially, it takes a starting value `b`, and a function that returns `Nothing` in its base case (when nothing more should be added to the list), and `Just (a, b)` in the recursive case. The first part of the tuple, `a` represents what should be added to the list, while the second part, `b` reprents the *remainder* of the value. 

I admit, that sounds complicated, but an example should help. For `fromDecimal n`, the base case is when $n$ is $0$. Here, we add nothing to our result list. In the recursive case, we add $n \mod 10$ to the list, since this gives us the leading digit of $n$, and continue with $\lfloor n / 10 \rfloor$ as our remainder.

Finally, following precisely from this definition, in Haskell [^2]:

```hs
fromDecimal = unfoldr $ \case
  0 -> Nothing
  n -> Just (n `mod` 10, n `div` 10)
```

With this, we can write our function, `add`:

```hs
add xs ys = fromDecimal (toDecimal xs + toDecimal ys)
```

For those counting, that was just six or so declarative lines of code, but depending on how comfortable you are with unfolds, that may have made very little sense. Still, unfolds are a powerful and general construct which can be useful in many situations, so it is good to be familiar with them. Anyway, next question.

## [Merge Intervals](https://leetcode.com/problems/merge-intervals/)

*Given an array of intervals where `intervals[i]` ==  `(start_i, end_i)`, merge all overlapping intervals, and return an array of the non-overlapping intervals that cover all the intervals in the input.*

In this question, we are given as input a list of `(Int, Int)` tuples representing the start and end of an interval. Our task is to merge the overlapping intervals in the list together. For example

```hs
merge [(1, 3), (2, 6), (8, 10), (15, 18)]
 = [(1, 6), (8, 10), (15, 18)]
```

Here, the end of the first interval extends past the start of the second, so we merge those two together into the interval `(1, 6)`. There is no overlap with the other intervals, so they stay put.

Our first step will be to sort the list by the first element of the tuple, so that possibly-overlapping intervals will sit neatly in sequence. We can do this simply with `sortOn fst`.

Next we want to iterate over our sorted list of tuples with an accumulator list we'll call `merged`. Our base cases are simple enough: if `merged` is empty, we just put the first tuple into merged and continue on, and if our input is empty we return `merged`.

The recursive case is a bit trickier. Here we will compare the first elements of `merged` and our input. If the *end* of the last-merged interval extends past the *beginning* of our next interval, we merge the two by prepending a tuple of the beginning of the last-merged, and the max of the ends of the two tuples. This ensures that overlapping intervals are collapsed together. If they do not overlap, we simply continue on by prepending the next tuple onto our merged list.

Here's my annotated solution looks like in Haskell:

```hs
merge :: [(Int, Int)] -> [(Int, Int)]
merge = go [] . sortOn fst -- sort then recurse
  where
    go merged [] = reverse merged -- base case: input is empty
    go [] (x : xs) = go [x] xs -- base case: merged is empty
    go ((a, b) : merged) ((c, d) : xs) =
      if b >= c -- if overlapping
        then go ((a, max b d) : merged) xs -- collapse intervals
        else go ((c, d) : (a, b) : merged) xs -- add new interval, leave old
```

You may notice that this pattern of recursion over a structure with an accumulator is eerily reminiscent of a fold, but in this case I couldn't figure out how to translate this `go` function into a fold, since we have the added complexity here of needing to alter our accumulated list as we proceed. If there is a way to do this with a fold (or maybe using [recursion-schemes](https://hackage.haskell.org/package/recursion-schemes)?) that I'm missing, please let me know!

## [Group Anagrams](https://leetcode.com/problems/group-anagrams/)

*Given an array of strings, group the anagrams together. You can return the answer in any order.*

In this question, we are given a list of strings, and must group the *anagrams* in the list, i.e. the ones that have all the same characters, yet in a possibly different order. For example, we should have:

```hs
groupAnagrams ["eat", "tea", "tan", "ate", "nat", "bat"]
  = [["bat"],["eat","tea","ate"],["tan","nat"]]
```

As a first step, it might be useful to review how to determine if two strings are anagrams. Since the only requirement is that they have the exact same characters, we can just convert both strings to sets, then compare the sets for equality, like so:

```hs
import qualified Data.Set as Set

-- ex: anagrams "cat" "tac" == True
anagrams :: [String] -> [String] -> Bool
anagrams s1 s2 = Set.fromList s1 == Set.fromList s2
```

Then, in order to group together anagrams, all we would need to do is sort the list according to their set representation, then group together anagrams. The functions `sortBy` and `groupBy` in `Data.List` help us do exactly this, as in the solution below:

```hs
import Data.List (sortBy, groupBy)
import Data.Function (on)

groupAnagrams :: [String] -> [[String]]
groupAnagrams =
  groupBy anagrams . sortBy (compare `on` Set.fromList)
```

This is a pretty clean one-liner, but it may not be the most efficient, with a time complexity of $O(m \cdot \text{log}(m) \cdot n \cdot \text{log}(n))$, where $m$ is the length of the strings, and $n$ is the number of strings. This complexity comes from the $O(n \cdot \text{log}(n))$ time to sort the string sets, and then $O(m \cdot \text{log}(m))$ to convert each of the strings to sets.

Can we do any better? Maybe we can if we ditch sorting, and instead use a dictionary. We could, for instance, create a `Map` from string-sets as keys to lists of strings as values. For each string, we insert it into the map at the key of its set representation, appending it to a list of those already seen. Then to extract the groups of anagrams, we just return a list of the values from this map. Here's the idea in Haskell:

```hs
import qualified Data.Map as Map

groupAnagrams' :: [String] -> [[String]]
groupAnagrams' = Map.elems . foldr insert Map.empty
  where
    insert s = Map.insertWith (++) (Set.fromList s) [s]
```

This actually has the same time complexity, since Haskell's `Data.Map` is a persistent data structure with $O(\text{log}(n))$ time cost for insertion, and we still have to convert each string to a set, which also takes $O(\text{log}(m))$ time. However, if this were performance critical, there's more optimizations we could make here with faster hashmap implementations, and my gut says this version would be faster (though I haven't done benchmarking, so who knows!)

## [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/)

*Given an integer array nums, return an array answer such that `answer[i]` is equal to the product of all the elements of nums except `nums[i]`. You must write an algorithm that runs in $O(n)$ time and without using the division operation.*

This next question asks us to map an array of integers to one where each element is equal to the product of the original array, excluding itself. For example, we should have

```hs
productExceptSelf [2, 3, 4] = [12, 8, 6]
```

As the prompt suggests, if we were allowed to use division, the problem would be trivial. We could just compute the overall product, then map the list, dividing the product by each element:

```hs
productExceptSelfDivision :: Fractional a => [a] -> [a]
productExceptSelfDivision nums = 
  let p = product nums
   in map (p /) nums
```

But with the restriction to integers, we are going to have to be a bit more clever. The key insight is that we can use the incremental products from the left and the right sides in order to compute the product *without* the current element.

By incremental products, I mean the list of products we generate by starting at $1$ and multiplying each element by the previous product. In Haskell, this "scanning" function is in the standard `Data.List` library as `scanl` and `scanr` for left and right scans respectively. These functions, similar to a fold, take a binary function, a starting value, and a list to iterate over.

Lets look at the left and right product scans for our example list:

```hs
let nums = [2, 3, 4]
scanl (*) 1 nums 
  = [1, 2, 6, 24] -- [1, 1 * 2, 1 * 2 * 3, 1 * 2 * 3 * 4]
scanr (*) 1 nums 
  = [24, 12, 4, 1] -- [1 * 4 * 3 * 2, 1 * 4 * 3, 1 * 4, 1]
```

With each, we start at $1$, multiplying by the previous product, arriving at the total product as the last element in the left scan, and the first element in the right scan.

Looking at the left scan, we might notice that *except for the last element*, each element represents the incremental product from the left *without* the contribution from the element at the same index in the original list. For example, consider comparing the two lists:

```hs
let original = [2, 3, 4]
let leftProducts = [1, 2, 6] -- incremental products, except last
```

Here, `leftProducts[i]` holds the incremental product without the contribution of `original[i]`. The same pattern holds when we compare every element *except the first* from the right scan to the original, such as here:

```hs
let original  = [2, 3, 4]
let rightProducts = [12, 4, 1]
```

Same thing, `rightProducts[i]` is the incremental product without `original[i]`. 

We can utilize these two facts to solve our original problem. To find the *total* sum at an index without the contribution of that position's element in the original list, all we need to do is to multiply element-wise the left and right scans, excluding the last element from the left and the first element from the right. 

When arranged in this way, both side's incremental products excludes that position's contribution, so multiplying them together gives us the total product with the same exclusion.

In Haskell, our final solution is just this:

```hs
productExceptSelf :: [Int] -> [Int]
productExceptSelf nums = zipWith (*) leftProducts rightProducts
  where
    leftProducts = init (scanl (*) 1 nums)
    rightProducts = tail (scanr (*) 1 nums)
```

Here we use `init` and `tail` to extract the parts of the respective scans that we want, then multiply these together with `zipWith`. Additionally, this implementation runs in $O(n)$ time.

## Conclusion

So there are my Haskell solutions to these four Leetcode questions. I focused on array/list based questions in this one, but Haskell also excels at solving tree-based questions, with its support for algebraic data types and pattern matching.

I hope you found these solutions interesting, or if you see a mistake or an improvement in any of these, please feel free to reach out via [Twitter](https://twitter.com/micah_cantor) or [email](mailto:hello@micahcantor.com).

------

*If you are hiring a software engineer intern for the summer of 2022, please [reach out](mailto:hello@micahcantor.com)!*


### Notes

[^1]: It shouldn't be too surprising that `unfoldr` is useful here, given that `zipWith` and `sum`, which we used in the complementary function, are really just specialized versions of `foldr`.

[^2]: I'm using the [LambdaCase](https://typeclasses.com/ghc/lambda-case) language extension here.