+++
title = "Crafting Interpreters in Typed Racket"
date = 2021-11-22
description = "Some thoughts on following Bob Nystrom's book Crafting Interpreters in Typed Racket."
[taxonomies]
tags = ["racket", "programming-languages"]
+++

This fall, I followed Bob Nystrom's excellent book on writing interpreters, [Crafting Interpreters](https://craftinginterpreters.com/), in [Typed Racket](https://docs.racket-lang.org/ts-guide/). The first part of Nystrom's book details how to write a tree-walking interpreter for a small, dynamically-typed language called [Lox](https://craftinginterpreters.com/the-lox-language.html), which supports classes and higher-order functions.

I chose not to follow along in Java for two reasons: first, I just don't like programming in Java, and second, I thought I'd get more out of the book if I forced myself to think through the translation of each line of code. So instead, I chose to use Typed Racket, which is the gradually-typed sister language to [Racket](https://racket-lang.org/). Initially, I started with just plain Racket, but especially since I was translating Nystrom's code, I kept running into runtime type errors, so I decided to give Typed Racket a try instead. I already had experience with most of Typed Racket's main features, namely algebraic data types and parametric polymorphism, from my experience with Haskell, so the jump was not too big.

Despite this, my main focus with this interpreter was not to make full use of Racket's language-oriented tools like macros, since I wanted to focus at first at understanding how to write an interpreter with a more standard approach. Still, in this post, I'd like to go over some of the interesting parts about the design of my interpreter and the notable differences from Nystrom's version.

If you are interested, the full code for the interpreter can be found [here on GitHub](https://github.com/micahcantor/racket-lox).

## Scrapping the Visitor Pattern

Nystrom's Java implementation makes use of [the Visitor pattern](https://craftinginterpreters.com/representing-code.html#the-visitor-pattern) to approximate a functional style in Java. The Visitor pattern is used in the interpreter to apply different evaluation methods for different AST nodes. However, as hinted at in the section about [the expression problem](https://craftinginterpreters.com/representing-code.html#the-expression-problem), this problem can be solved cleanly in functional languages by instead using [sum types](https://en.wikipedia.org/wiki/Tagged_union) and pattern matching.

So, rather than creating overloaded visit methods for each AST nodes, I instead have a single `evaluate` function which takes an `Expr` type and then uses a `cond` to select which kind of expression we are evaluating in order to call the appropriate function:

```racket
(: evaluate (-> Interpreter Expr Any))
(define (evaluate i expr)
  (cond
    [(literal? expr) (eval-literal i expr)]
    [(variable? expr) (eval-variable-expression i expr)]
    [(assign? expr) (eval-assign i expr)]
    [(grouping? expr) (eval-grouping i expr)]
    [(unary? expr) (eval-unary i expr)]
    [(call? expr) (eval-call i expr)]
    [(get? expr) (eval-get i expr)]
    [(set-expr? expr) (eval-set-expr i expr)]
    [(super-expr? expr) (eval-super-expr i expr)]
    [(this-expr? expr) (eval-this-expr i expr)]
    [(binary? expr) (eval-binary i expr)]))
```

Here, `Expr` is defined simply as a union type of all of the different kinds of expressions, so this evaluation function is exhaustive. This results in cleaner code, and is in my opinion one of the main reasons to choose a functional language over an OOP one for writing an interpreter.

My Racket version also needs much less code to define each AST node type, as each node is simply a Racket struct containing the appropriate fields. For example, the `binary` expression struct looks like this:

```racket
(struct binary ([left : Expr] [operator : Token] [right : Expr]))
```

Therefore, we have no need for [the metaprogramming that Nystrom applies](https://craftinginterpreters.com/representing-code.html#metaprogramming-the-trees) to automatically generate the class files for each verbose AST node. 

## Implementing Callable

Nystrom creates an interface, `Callable`, to abstract over a few different classes that all implement a `call` function in slightly different ways: functions, native functions, and classes. In our functional paradigm though, we don't have a direct corrolary to OOP interfaces.

Instead, we will reuse the same pattern from the prior section by defining `Callable` as a union type of the different options, `Function`, `NativeFunction` and `Class`:

```racket
(define-type Callable (U Function NativeFunction Class))
```

Then, rather than implementing separate `call` functions for each type, we have a single `callable-call` function that uses pattern matching to select which type of call to perform, like so:

```racket
(: callable-call (-> Callable Interpreter (Vectorof Any) Any))
(define (callable-call callee i args)
  (match callee
    [(function _ _ _) (call-function callee i args)]
    [(native-function call-func _) (call-func callee i args)]
    [(class _ _ _) (call-class callee i args)]))
```

In this case, I would say the sum type and pattern matching pattern is no cleaner than the Java interface approach, but it is a good example of a simple and common design pattern used in functional programming.

## A few helper macros

Although I did not use them extensively, given that Racket is a Lisp, I made use of a few utility macros to make a few parts of my code cleaner.

### While Loops

Since I followed Nystrom and often wrote in an imperative style, it helped to have a while loop construct for repeatedly performing side-effecting code. Racket has no built in while loop, but we can write one easily with a simple macro:

```racket
(define-syntax-rule (while pred? body ...)
  (let loop : Void ()
    (when pred?
      body ...
      (loop))))
```

Here we define a simple syntax rule that replaces an instance of `(while pred? body ...)` with a 'named let' loop function that calls itself recursively as long as `pred?` is true.

This is helpful, for instance, in our evaluation of Lox's while statements, which is simply:

```racket
(: exec-while-stmt (-> Interpreter WhileStmt Void))
(define (exec-while-stmt i stmt)
  (match-define (while-stmt condition body) stmt)
  (while (truthy? (evaluate i condition))
         (execute i body)))
```

As long as the while statement's condition is truthy, we execute the body statement.

Here I also use the `match-define` procedure from `racket/match` to destructure our `stmt` object, a pattern I use frequently in the interpreter.

### Changes in Scope

One piece of our interpreter is the resolver, which makes a static analysis pass over the AST to resolve local variables before we evaluate the program. In the resolver, we carefully manage a stack of scopes that each contain a hash table with local bindings for that scope. Whenever we enter a block or function declaration, we push a new scope onto the stack, and then pop it off when we leave.

Translating directly from Nystrom's Java, originally the code to resolve a block statement looked like this:

```racket
(: resolve-block-stmt! (-> Resolver BlockStmt Void))
(define (resolve-block-stmt! r stmt)
  (begin-scope! r)
  (resolve-all! r (block-stmt-statements stmt))
  (end-scope! r))
```

We push on a scope with `begin-scope!`, resolve all the statements in the block, then pop off the scope with `end-scope!`. This isn't bad, but since we reuse this pattern in a few other places, we can make it a bit cleaner by introducing a macro, `with-scope`:

```racket
(define-syntax-rule (with-scope r body ...)
  (begin
    (begin-scope! r)
    body ...
    (end-scope! r)))
```

This simple macro wraps some given body statements with calls to begin and end the scope. Applying this to our block resolver above, we get this:

```racket
(define (resolve-block-stmt! r stmt)
  (with-scope r
    (resolve-all! r (block-stmt-statements stmt))))
```

Not a huge change, but it does make the code a bit less repetitive.[^1]

## Conclusion

So, those were a few scattered thoughts on this little Racket interpreter. Overall, I enjoyed the experience of working with Typed Racket and learning the basics of interpretation from Nystrom. 

As of now, according to [this list](https://github.com/munificent/craftinginterpreters/wiki/Lox-Implementations#racket), it looks like I am the only one who has implemented Lox in Racket, but I'm glad I was able to add one of my favorite languages to that list and learn a bit along the way.

------

*If you are hiring a software engineer intern for the summer of 2022, please [reach out](mailto:hello@micahcantor.com)!*

### Notes

[^1]: I stole this idea from [this implementation of Lox in Chicken Scheme](https://github.com/harryposner/schlox). If you found this post interesting, you may find their code to be as well.


