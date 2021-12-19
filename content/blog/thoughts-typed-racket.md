+++
title = "Typed Racket: Good and Bad"
date = 2021-12-19
description = ""
[taxonomies]
tags = ["racket", "programming-languages"]
+++

There's a lot to like about [Typed Racket](https://docs.racket-lang.org/ts-guide/), which adds a powerful yet flexible type system to the [Racket language](https://racket-lang.org/). [In my last post](/blog/crafting-interpreters-typed-racket/), I wrote about how I used Typed Racket (TR from now on) to implement a tree-walking interpreter for [the Lox language](https://craftinginterpreters.com/the-lox-language.html) from [Crafting Interpreters](https://craftinginterpreters.com/).

Along the way, I gathered some thoughts about my chosen implementation language that I'd like to divulge here. Despite some good aspects of TR, even in this medium-sized project, I became frustrated with parts of TR's UX through its tooling and documentation. I'll break this up into two sections: the good and the bad parts of my experience. Let's start with the good.

## The Good

### Building on Racket

Typed Racket builds on the Racket platform through the [#lang feature](https://beautifulracket.com/explainer/lang-line.html). This means that Typed Racket code is transformed into plain Racket after type-checking, and therefore can make use of any Racket libraries or interface with existing Racket code. Racket is a mature and stable project with excellent documentation and an extensive standard library, and though it may have fewer third-party libraries than other more popular languages, I still think it is a great foundation to build a language on.

### Migration and Interoperability

With that in mind, converting from Racket to Typed Racket is not very difficult. Essentially all you need to do is change the `#lang` line to `typed/racket` and then add type annotations to function signatures, struct defintions, and a few other places like mutable values.

The story for typed-untyped interoperability is also good. TR comes built-in with typed versions of most (all?) of the standard library. Additionally, any untyped code can be imported with the `require/typed` form, where you essentially just need to add type signatures to the data types and functions you import.

### Algebraic Data Types and Polymorphism

The main reason why I chose TR is that on top of being a typed Lisp, it crucially has support for [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type) (ADTs) and [parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism) (generics). These two features are relatively small compared to a full OOP/interface setup, yet are powerful enough to allow you to write extensible code for a variety of applications. ADTs specifically, and having proper [sum/union types](https://en.wikipedia.org/wiki/Tagged_union), are something I find to be hard to give up once you have used them in any language.

### Occurrence Typing and Inference

Typed Racket is distinguished by its relatively uncommon type system based on [occurence typing](https://docs.racket-lang.org/ts-guide/occurrence-typing.html). Essentially this allows the TR to resolve types based on whether a predicate passes or fails on a particular value. Here's an example directly from their documentation. Suppose we want a function `flexible-length` that returns the length of either a string or a list. We can write

```rkt
(: flexible-length (-> (U String (Listof Any)) Integer))
(define (flexible-length str-or-lst)
  (if (string? str-or-lst)
      (string-length str-or-lst)
      (length str-or-lst)))
```

Here, we annotate `str-or-list` has being a union type of either a `String` or `(Listof Any)`. Then in the body we check if the value is a string with the predicate `string?`. Based on that check, TR knows that in the truthy branch of the `if` expression the value must be of type `String`, and in the falsy branch, it must have type `(Listof Any)`, so the uses of `string-length` and `length` each type-check respectively.

This ability to distinguish between types at runtime, yet still rely on type-checking at compile time is really powerful, and allows TR code to be quite dynamic yet still type-safe.

Built on top of this occurence typing, type inference for TR is usually good, meaning that type annotations outside function signatures are needed only for mutable values (whose type could change), or some higher-order polymorphic functions.

## The Bad

That all being said, there are still some difficulties and frustration I had with TR that I found while using it to write the interpreter.

### Slow Compilation Times

With TR's type-checking, I of course expect there to be reasonable compile times, but the TR compiler is still slow. On my laptop with a Ryzen 4500u, compiling about 1500 lines of TR from my interpreter to bytecode with `raco make` takes 14.2 seconds. TR does cache some compilation results, so subsequent calls are faster, but from unscientific benchmarking, just changing a comment in one file still takes 7.7 seconds to recompile after caching.

In comparison, on my machine, compiling the around 2000 lines of Java that make up Nystrom's original implementation of Lox takes just 2.4 seconds. Of course a comparison like this is not at all one-to-one, and I don't expect a small compiler like TR to have the same optimizations or speed as the Java compiler, but it is nonetheless something to keep in mind when choosing an implementation language, especially for a large project.

### Option vs. Opt

In Racket, there is only one falsy value, i.e only one that forces the else-branch of an `if` expression: the literal `#f`. In contrast, every other Racket value is truthy. In dynamic Racket code, we often use `#f` as a stand-in for empty values, similar to returning `null` in Java. For example, the list find procedure `findf` returns `#f` if  the desired value is not in the list.

In TR this concept is cemented with the built-in `Option` type, which is defined as the union of a type and `False`:

```rkt
(define-type (Option a) (U a False))
```

where `a` is a generic type parameter.

Although it can be convenient, this pattern sometimes lead to type-unsafe code, since we are 'doubling up' on the usage of the `False` type as both an empty value and a boolean. For example, say you have a hash table that associates strings to booleans, like so:

```rkt
(define names : (HashTable String Boolean) (make-hash))
(hash-set! names "John" #f)
```
In Racket, you can look up a value in this table with `hash-ref`, and then specify `#f` as an empty value to return if the key is not in the table, such as in the following:

```rkt
(hash-ref names "John" #f)
(hash-ref names "Jane" #f)
```

The problem with this approach is that we actually can't tell the difference between whether `hash-ref` succeeded (as it would for `"John"`), and the value associated with the key is `#f`, or if the key was not in the table, and thus returned `#f`, as it would for `"Jane"`. This example might seem silly, but it is actually a bug that I ran into while writing the interpreter when I was careless about handling a hash table with boolean values.

A solution to this and similar issues would be to define the optional type differently that doesn't conflict with booleans. Indeed, in the TR guide, they show how to define a `Opt` type similar to the `Maybe` type in Haskell:

```rkt
(struct None ())
(struct (a) Some ([v : a]))
(define-type (Opt a) (U None (Some a)))
```

This creates a union type `Opt` which is either an empty value `None`, or a generic type wrapped in the type constructor `Some`. By using `Opt` to specify empty values, we can avoid the bug described above by, say, returning `None` as the default value passed to `hash-ref`.

The problem I have with TR, is that the confusingly named `Option`, the one where `#f` is the empty value, is the type provided in the standard library. This other `Opt` type is given as a a prominent example of how to write polymorphic data types in the [Typed Racket guide](https://docs.racket-lang.org/ts-guide/types.html#%28part._.Polymorphic_.Data_.Structures%29), but its use (or at least documentation about the unsafety of `Option`) is not codified in the language. 

I think that TR should clarify this confusing relationship between `Option` and a user-defined type like s`Opt` more clearly and give some guidance on when to choose one or the other.

### Typing Hash Tables

This is small, and I'm not sure if I'm just missing something, but speaking of confusion with hash tables, I can't seem to get the `hash-ref` procedure to type-check correctly. Consider the following code snippet where we lookup a integer value in our names table with string keys:

```rkt
(define names : (HashTable String Boolean) (make-hash))
(hash-ref names 5)
```

This just returns a runtime error, no value found for the key `5`, but it should instead be a compile time type error, since `5` is clearly not a string. Again, if I'm doing something wrong here please reach out, because otherwise I don't see why this isn't a bug in the type of `hash-ref`.

### Inconsistent Documentation

The Racket language has excellent documentation, both in the form of the extensive [Racket Guide](https://docs.racket-lang.org/guide/) and the even more detailed [Racket Reference](https://docs.racket-lang.org/reference/index.html). Typed Racket's [guide](https://docs.racket-lang.org/ts-guide/index.html) and [reference](https://docs.racket-lang.org/ts-reference/index.html) are good, but more inconsistent. I already talked a bit about the confusion between the `Option` type and similar user-defined types, but there are other issues as well.

For one, there is inconsistency in whether polymorphic type variables should be named as uppercase or lowercase letters. As we saw earlier, in the section of the guide [Polymorphic Data Structures](https://docs.racket-lang.org/ts-guide/types.html#(part._.Polymorphic_Data_Structures)), `Opt` is defined with lowercase type variables: 

```rkt
(define-type (Opt a) (U None (Some a)))
```

but in the very next section, [Polymorphic Functions](https://docs.racket-lang.org/ts-guide/types.html#%28part._.Polymorphic_.Functions%29)), a function type is written with uppercase type variables:

```rkt
(: list-length (All (A) (-> (Listof A) Integer)))
```

I don't really care which is used but the documentation should stick to one style or the other.

There's a similar issue in whether to use the procedure type constructor `->` as infix or prefix. In most parts of the documentation, the prefix notation is used, so the type of a procedure from `String` to `Number` would be `(-> String Number)`. In [other parts](https://docs.racket-lang.org/ts-guide/occurrence-typing.html#%28part._let-aliasing%29) however, the same type is written using infix notation as `(String -> Number)`. The fact that both of these create the same type and are used interchangeably is a bit confusing, given that everything else in Racket is generally written with prefix.

Lastly, it would be nice if the guide talked more about how to define one's own type predicates in the [Occurence Typing](https://docs.racket-lang.org/ts-guide/occurrence-typing.html) section. For example, at one point, I wanted to write a predicate to assert that a list contained only strings. I couldn't find an example anywhere about how to actually do this, but helpfully found out from [this StackOverflow answer](https://stackoverflow.com/questions/69694263/how-to-assert-listof-string-predicate-in-typed-racket).

This assertion can be defined as such:

```rkt
(: (-> Any Boolean : (Listof String)))
(define (listof-string? (l : (Listof Any)))
  (andmap string? l))
```

I think a simple example of somthing like this ought to be found somewhere in the documentation, given that it wasn't entirely clear that I could define and compose my own predicates in this way.

### Editor Integration

The choice of editors for TR is not great. There is of course DrRacket, the built-in IDE, but DrRacket is designed to be used by students who are new to programming, and is just not nearly as smooth or polished of an editing experience as I'm used to with VSCode.

The other option is Emacs, which has long been the go-to editor for lisps. Emacs has [racket-mode](https://www.racket-mode.com/) as well, which provides smart editor features for Racket. But I have no idea how to use Emacs proficiently, and when I looked into it, I realized I would need about as much time to learn Emacs as I would to just write the interpreter, so I didn't bother. Maybe I should just bite the bullet if I want to write more Racket in the future though.

So, instead of those options I used VSCode with the [Magic Racket](https://marketplace.visualstudio.com/items?itemName=evzen-wybitul.magic-racket) extension and the [Racket language server](https://github.com/jeapostrophe/racket-langserver). This setup *works*, it gets you syntax highlighting, and the language server will usually tell you when there is a syntax or type error. But it is still not at all a great editing experience for a bunch of minor reasons. Magic Racket's syntax highlighting is not entirely consistent, and the language server basically crashes whenever I add a new import. There is also currently no support for displaying TR type signatures on hover.

None of these are deal-breakers, but they add up to an editing experience in VSCode that is well behind other functional languages like Clojure with [Calva](https://calva.io/) and [Paredit](https://calva.io/paredit/) or Haskell with [HLS](https://github.com/haskell/haskell-language-server).

Of course the community for Typed Racket is much smaller than for either of those languages, but personally I think that ergonomic tooling and editor integration is the kind of work that isn't always sexy, but will help retain users of the language.

## Conclusion

All in all, I liked learning and using Typed Racket in [my Lox interpreter](https://github.com/micahcantor/racket-lox). It takes an already good language in Racket and adds robust static typing. However, that didn't mean there weren't some frustrations along the way. I hope that some of these issues can be improved on and that Racket and its typed sibling find its way to more programmers in the future.

If you are a more experienced user of Typed (or untyped) Racket and you have question or comment, please feel free to reach out via [Twitter](https://twitter.com/micah_cantor) or [email](mailto:hello@micahcantor.com).