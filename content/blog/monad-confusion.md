+++
title = "Monad Confusion and the Blurry Line Between Data and Computation"
date = 2022-07-27
description = "The dual interpretations of data and computation, and a comparison of Clojure macros to Haskell monads."
[taxonomies]
tags = ["haskell", "clojure", "functional-programming"]
+++

There's a common joke that the rite of passage for every Haskell programmer is to write a "monad tutorial" blog post once they think they finally understand with how they work. There are enough of those posts out there, though, so I don't intend for this to be yet another monad tutorial. However, based on my learning experience, I do have some thoughts on *why* people seem to struggle so much with monads, and as a result, why so many of those tutorials exist.

At a high level, the intuition for monads are that they are an abstraction of sequencing in programming. Any computation that involves "do this, and then do that using the previous result" can be considered monadic. 

For some types that implement `Monad`, what I think of as "computational monads" like `IO` or `State`, this intuition make sense. We tend to think of these types as computations since they represent verbs like "performing input/output operations" or "computing a value while tracking state." In this sense, we interpret their monad instances as describing how two of these computations can be sequentially composed together.

I think many Haskell newcomers struggle to understand how monads work comes when trying to apply this intuition to a group of seemingly unrelated types that I'll call "data monads." These are ones like lists and `Maybe`, which we typically think of as representing nouns, namely "a list of values" or "an optional value". The problem is this: how can these data types be sequenced together? How are they monads?

Fundamentally, I think this issue stems from the fact that monads blur the line between *data* and *computation*, even though we usually think of these two to be distinct concepts. I argue that the trick to understanding data monads is to interpret them instead as computations: as verbs rather than as nouns.

This blurry line between data and computation, however, is not unique to monads. In fact, it's exactly that blurry line that lies at the core of functional programming: first-class functions.

## Computation as Data

First-class functions, or the ability to represent functions as ordinary values, are a core aspect of the functional programming paradigm. A language with first-class functions allows any function to return another function or to take a function as an argument. 

They also precisely represent the blurriness between data and computation. We normally think of a function as a computation: something that takes some input and computes an output. But with first-class functions, we can instead use functions as data — as objects that may be passed around like a string or an integer.

But functions are not the only type of computation, or at least, we often work with more complicated kinds of computation like `IO` and `State`. These are types that compute a value while simultaneously tracking other effects outside of the normal input and output of a function. 

Part of what makes Haskell's type system so powerful is its ability to encode these other types of computation as data. In fact, if we look at a basic definition of `State`, we see it's really just a generic wrapper over a specific function signature:

```hs
newtype State s a = State (s -> (a, s))
```

That is, the type `State s a` is a function that takes an initial state as input, and returns an updated value and state. By wrapping this function in a data type, we can more easily work with the stateful computation as data. That is, we can use it as input or output to a function, or we can wrap it in some other computation, which is what the more practical `StateT` monad transformer does.

In general, the technique of treating computations as data types is pervasive in Haskell code. It allows one to scope and isolate side-effecting code like `State` or `IO` from pure code and to compose multiple effectful computations together.

So what *is* a monad? All monads represent some computation, i.e. a verb or action, that can be sequentially composed together and may carry with it some side effects. This is sometimes unclear, however, since we often work with these actions as if they were ordinary data types.

## Data as Computation

So what about those "data monads" like lists and `Maybe`? They can be interpreted as computations too, and their `Monad` instances allow us to tap into a rich set of generic tools to work with them as such.

I think an example will help explain this idea better, so let's look at representing data as computation with the `Maybe` monad. As I said earlier, we traditionally think about `Maybe a` as a generic data type that represents either the presence of that data (with `Just a`), or the absence of that data (with `Nothing`).

But the monad instance of `Maybe` exploits a different interpretation of the same type. Instead, `Maybe a` can be seen as a computation with a side-effect, namely that in the process of computing `a`, the computation may return nothing. If it does, it returns `Nothing`, and otherwise it returns `Just a`. 

If you consider `Maybe` as a computation, then the monad instance for the type may make more sense. Remember that the implementation for the monadic bind operator (`>>=`) tells us how to sequentially execute two computations while threading the inner value along. We can implement monad for `Maybe` like this:

```hs
instance Monad Maybe where
  m1 >>= m2 = case m1 of
    Just x  -> m2 x
    Nothing -> Nothing
```

This instance tells us how to string together two `Maybe` computations, `m1` and `m2`: If the first computation returns a value (`Just x`), then the sequence returns the result of the next computation wrapped around that value (`m2 x`). Otherwise, if the first computation returns `Nothing`, then the sequence also returns `Nothing`. 

Essentially, this implementation means that a sequence of `Maybe` computations passes the inner value along the chain unless an intermediary computation returns `Nothing`, in which case the whole sequence computes `Nothing`. So while it may be difficult to understand how a piece of data like `Maybe` can be sequenced, if we instead consider `Maybe` as a computation, the question of how to sequence it becomes more intuitive.

This is of course just one example, and the implementation of bind for `Maybe` is a particularly simple one, but I think it illustrates an important point: when trying to understand the monad instance of a type, first try to understand what kind of computation that type represents, and then tackle how two of those computations may be sequenced.

A similar data-computation parallel exist for the list monad, where we consider lists as representing a non-deterministic computation, or one that explores all possible paths of execution and collapses the values from each. This parallel interpretation of lists may better help understand how to implement its monad instance.

## Conclusion

In summary, Haskell allows for and encourages the interplay between data and computation. It's type system enshrines all computations as first-class values, enabling rich representations of computation as data. Meanwhile, monads provide a generic set of tools for working with data types as computations that may be sequenced and chained together.

Grasping these two concepts has been crucial to my understanding of how to work with Haskell's monads, so maybe this can help you as well. As always, feel free to reach out [on Twitter](https://twitter.com/micah_cantor) or by [email](mailto:hello@micahcantor.com), I'd love to talk more about this topic.

## Addendum: Lisp Macros and "Code is Data"

As I thought more about this blurry line in Haskell between data and computation, it reminded me of a similar concept from the Lisp family of languages. In a Lisp, the syntax of s-expressions allows code to be handled as data, which enables another form of the computation as data technique: macros.

In fact, there's a direct parallel that can be made between Lisp macros and monadic code in Haskell, which often resemble each other in their usage and implementation.

Take for instance, the `when` macro in Clojure, which is defined like this:

```clojure
(defmacro when [test & body]
  `(if ~test
    (do ~@body)
    nil))
```

If you are unfamiliar with Lisp macros, this defines `when` as a macro that expands to a simpler `if` expression. In that expression, if the test is true-ish then the body is evaluated, and otherwise it returns `nil`. 

The macro can be used to conditionally execute some imperative code, for instance you may write

```clojure
(when engines-ready
  (clear-surroundings)
  (launch-rocket))
```

which does nothing if `engines-ready` is `false`, and otherwise executes the body.

An extremely similar construct is provided in Haskell, yet it is defined completely differently: not as a macro but as a function over a `Monad`[^1]:

```hs
when :: Monad m => Bool -> m () -> m ()
when test body =
  if test 
    then body 
    else pure ()
```

This is a function with two arguments: a boolean test and a monadic action. If the test is `True` we evaluate the action, and otherwise we "do nothing", by returning the dummy unit value wrapped in the monad with `pure ()`. In Haskell we can translate the same piece of code as

```hs
when enginesReady $ do
  clearSurroundings
  launchRocket
```

Comparing the two definitions between languages we can see that where the Clojure macro takes a piece of code as input, the Haskell function instead takes a monadic action. In this sense, both systems allow the programmer to treat any computation, and not just functions, as first-class values.

The difference however, is that Haskell's approach encodes this information in its typechecker, whereas the Clojure macro is dynamically typed. In this case, that means that binding the result of `when` in Clojure can either give whatever the result of the body expression is, or it could bind to `nil` and eventually lead to a null pointer exception. 

In Haskell however, that would be impossible, since its `when` always returns `m ()`, ensuring a variable cannot sometimes take a value and sometimes be mistakenly bound to nothing. In fact, if we did want this behavior, we could enrich the type of `when` to instead return `m (Maybe a)`, giving us `whenMaybe`:

```hs
whenMaybe :: Monad m => Bool -> m a -> m (Maybe a)
whenMaybe test body =
  if test 
    then Just <$> body 
    else pure Nothing
```

Here, if the test is true we return the result wrapped in `Just`, and otherwise return `Nothing`, all within the monadic computation.

That's not to say Haskell's approach is strictly better however, since the overhead of managing the types of complicated monadic actions may not be worth the runtime safety it provides. 

I do think this is an interesting example though, since it shows how these two incredibly different languages both allow for the expression of a core aspect of functional programming, computation as data, but in radically different ways.

### Notes

[^1]: This function should actually be constrained instead using `Applicative` rather than `Monad`, since we only need to use `pure` and not bind. But all monads are applicative functors so this stricter version is fine for this example.