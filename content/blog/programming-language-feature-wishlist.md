+++
title = "My Programming Language Feature Wishlist"
date = 2023-07-09
description = "All the features I want in a modern general-purpose language"
[taxonomies]
tags = ["programming-languages", "compilers"]
+++

As I've gotten more experience using different programming languages, I've started to become more opinionated about which features I like and which I don't. In this post, I want to share my "wishlist" for features in a modern programming language. All these features can be found in some language, but I don't think a single language exists that combines all of them.

For this post, I want to focus specifically on features for a *general-purpose* language, not a scripting, systems, or domain-specific one. There are lots of interesting features eliminated by that focus, but this way the list will be more widely applicable. 

I also want to focus only on statically typed languages here. I think that dynamic type checking can be convenient, but for most large programs a static types are useful for organization and correctness. A few of the features on the list involve type systems so I thought it'd be easier not to repeat that preference each time.

After discussing the features I think are most important, I'll discuss a few candidate languages that come close and what they're missing in my view.

## The Wishlist

### Generic Algebraic Data Types

My language should support algebraic data types (ADTs) as the primary way for defining new data structures. ADTs generalize the pairing of `struct` and `enum` from C or Java with the ability to store data (like primitives or other ADTs) within each `enum` variant.

ADTs shine at defining robust utility data structures like `Option` and `Result` for error checking, or for recursive data structures like trees and linked lists. When combined with exhaustive pattern matching, functions that operate on ADTs become easy to write and maintain. Now, whenever I design a new program, I usually begin with designing ADTs to capture the structure of the inputs and outputs. Modern languages without them now feel clumsy and incomplete to me.

For increased flexibility, ADTs in my language should support type parameters. Parameterized types, also known as generic types, have become a standard feature in modern statically typed language. Nevertheless, my language's type system should include them since they fit particularly well with ADTs. 

Generic ADTs allow the user to define data structures that aren't pinned to a specific type. For instance, instead of needing to create distinct `IntList` and `FloatList` types as one would in a language without generics like C, we can create a single type `List a` that's parameterized by the type of list contents. This means we don't need to duplicate the `List` interface for each particular type, increasing code reuse.

### Type Constraints

When using generic types, it's natural to want a way to constrain the type of the parameter from *any* type to instead some subset. For instance, in Haskell we can express a sort function with the following type signature:

```hs
sort :: Ord a => List a -> List a
```

In words, `sort` takes a list of type `a` as input and outputs a list of type `a`. Importantly, it adds a constraint on the type of `a`, which Haskell calls a "typeclass" and Rust calls a "trait." The constraint forces `a` to be a type that can be ordered with some comparison operator `<=`. 

Constrained generics improve the correctness of functions like `sort` since it ensures at compile time that we can only sort types that implement `Ord`, rather than allowing any type and producing an incorrect result or a runtime error.

Another way to understand type constraints is in terms of dynamic dispatch. The comparison operator has a constrained generic type: 

```hs
(<=) :: Ord a => a -> a -> Bool
```

which takes two order-able elements and returns a boolean. However the implementation will differ based on the type of `a`. For instance, if `a` is an integer, the comparison will be the usual numeric comparison, but if `a` is a string, the operation will be alphabetic comparison.

Therefore, constraints allow for a form of dynamic dispatch, similar to interfaces in an object-oriented language like Java. However, constraints tend to be more lightweight and flexible than interfaces, since they're not limited to operating on classes as they are in Java, and don't need to account for the complexity added by OO-style inheritance.

### Parallel Runtime and Concurrency Primitives

My programming language's runtime should be designed with support for concurrent and parallel programs in mind from the beginning. OCaml recently underwent a years-long overhaul in the language's implementation to add backward-compatible parallelism to the language. Meanwhile others like Python and JavaScript continue to suffer from their language's dependence on their initial single-threaded focus.

My language should also provide some language support for asynchronous or non-blocking programming. The most popular paradigm for this problem right now is some primitive form of lightweight threads or coroutines, as found in Go, Lua and Kotlin. Other languages add syntactic support for labeling asynchronous functions with `async`/`await`, such as in C#, JavaScript and Rust.

Different languages take other approaches to these problems, and I'm not sure what is best. However, my language should decide on some concurrency primitives and syntax from the beginning. If they are added in later, then libraries tend to be split along lines of different concurrency conventions. For instance, in the JS, Rust, and Python communities, new libraries with different APIs needed to be created with the introduction of `async`/`await` causing users to "pick sides" on which libraries to use.

### Minimal C-like Syntax

I believe a language's syntax is relatively unimportant compared to its other features. However, if your language's syntax is significantly different from a standard C-like style, I think it will hurt its adoption and growth. For instance, see any online discussion about languages in the Lisp family and they tend to fixate on the "polarizing" syntax. A non-traditional syntax can distract from discussion and adoption of more important language features.

Even within a C-style syntactic framework, adding a lot verbose or novel syntax can also be difficult for newcomers. See Rust, for instance, which has added a lot of syntactic noise and cruft, adding to its difficulty. Therefore, my programming language should be thoughtful in its syntax design, but err on the side of tradition and minimalism when deciding between equivalent options. 

### Great Error Messages

My programming language should have error messages that are clear, concise, and human readable. Error messages should clearly and correctly outline:

- What went wrong;
- Where the error occurred; and
- Suggestions for fixing, debugging, or researching the error

A good error message aims to communicate these points both to beginners and experts. Where this balance is difficult, the message should default to being beginner-friendly, while including references or terms that makes it easy for experts to find additional information.

Importantly, the compiler authors should maintain that any error message that doesn't clearly communicate all of the above standards is a bug. Moreover, as error messages are one of the main ways users interact with the compiler, these bugs should be a top priority for maintainers to fix.

### Intuitive Installation and Tooling

Users expect a programming language, even ones that are new or just prototypes, to be easy to install and run. It shouldn't take longer than 5 to 10 minutes to go from the landing page to running "hello world" in my language. My language's command line tools should be fast, reliable, and clear when an installation or build has failed.

Additionally, programming communities have increasingly moved towards the language server protocol (LSP) as the primary way to enable editor tooling. My language should be designed with the LSP and compiler-as-a-service usage in mind from the beginning. The compiler pipeline should be modular and accessible from a library API, so, for instance, each community-driven tool doesn't have to re-implement a parser.

Supporting all the features of best-in-class tooling from mainstream languages is difficult and time-consuming, but I think focusing at first on ease-of-use will help drive support, adoption, and further development.

## Discussion and Contending Languages

There are several languages that in my view come close to matching this feature wishlist, a few of which I've already mentioned in this post. For each, I'll write about which features they include, which they don't, and some drawbacks and benefits I see in each.

### Haskell

Haskell has one of the most advanced type systems of any mainstream programming language, and I particularly like its syntax and implementation of constrained generic ADTs. Additionally, the GHC platform is an advanced, optimizing compiler with native support for concurrency and parallelism. 

Unfortunately, the language faulters along several other lines that I think harm its adoption, including unintuitive error messages and a tooling landscape that, though has seen great improvement, can be prone to bugs. I also think that although its syntax is elegant and minimal, it remains a barrier to entry for many users.

### OCaml

I view OCaml in many ways as a "pragmatic Haskell," in that it shares many of the same powerful type system features but compromises by not sandboxing imperative programming and using strict evaluation. The language also has recently seen increased investment and development, and OCaml 5.0, released this year, features a rewritten runtime to support parallel programs while preserving backward compatibility.

OCaml lacks an analog to Haskell's typeclasses, which I feel can often lead to clunky code duplication. This is partially accounted for by OCaml's support for effects, first-class and higher-order modules, all of which are intriguing features I considered including in this list. I also think its error messages are generally simpler than Haskell's and its tooling ecosystem more unified.

### Rust

Rust, in borrowing several features from Haskell and OCaml, has an advanced type system with constrained, generic ADTs. The language's defining feature, however is that it encodes memory lifetimes in its type system. This has led to its adoption as a modern replacement for C/C++, but its lack of garbage collection makes it difficult and tedious to use for programs that aren't performance-constrained.

Rust's error messages are certainly best-in-class and the quality of its main tooling system `cargo`, though it continues to receive active development, is often considered a standard in other language communities. However, its verbose syntax and focus on systems programming makes it difficult to embrace as a general purpose language.

### TypeScript

TypeScript has generic ADTs and has become a decent compromise language for developers looking for a modern type system features with a C-ish style syntax package. It also has the largest community of any of the languages I've mentioned so far, and with that a more developed tooling and package ecosystem. 

The V8 JavaScript engine can produce great performance, and new tooling innovations like [esbuild](https://esbuild.github.io/), [deno](https://deno.land/), and [bun](https://bun.sh/) will only advance TS/JS tooling further. However, TS has some warts and sharp-edges by building on top of JavaScript, and its focus on gradual typing and gradual adoption may limit the project's further innovation.

### Others

There are a few other languages I want to briefly mention that tick some, but not all, of these boxes:

- Elm: Generic ADTs, great error messages
- Scala: Advanced type system and FP support
- Roc: In development, with advanced type system and runtime
- ...many others!

## Conclusion

Overall, I think the future of programming language development is bright, and there are countless exciting projects seeing active development that are pushing the bounds of PL research and compiler implementation.

It may be frustrating that the features on my wishlist can be found in different languages but not in one place. But I think that may be a natural result of the fact that PL development is still relatively young as a discipline, and these ideas have surfaced from many projects with different goals and ideals. Additionally, compiler implementation is hard, and it takes skill, money, and time to build a new language with all of the features that users want.

As I mentioned at the beginning, this is purely my opinion of what features I look for in a language, but I'm curious to hear your thoughts on your feature wishlist or your favorite contending languages. If so, please feel free to reach out by [email](mailto:micahcantor01@gmail.com) or on [Twitter](https://twitter.com/micah_cantor).
