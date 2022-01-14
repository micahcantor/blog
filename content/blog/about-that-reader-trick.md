+++
title = "About that Reader trick"
date = 2022-01-14
description = "Reader can be used to track local bindings, but not all bindings."
[taxonomies]
tags = ["haskell", "programming-languages"]
+++

There is an interesting post on [Solomon's Blog](https://blog.cofree.coffee/2021-08-13-that-one-cool-reader-trick/) about how the [Reader](https://hackage.haskell.org/package/mtl-2.2.2/docs/Control-Monad-Reader.html) monad in Haskell can be used to model variable bindings in an interpreter.

For those unfamiliar with `Reader`, it represents a computation which can reference some environment and execute sub-computations in a modified environment. Concretely, this is often used in Haskell apps to reference some kind of initial configuration or context that's needed throughout. It would often be a pain to manually pass this environment to every function that requires it, so instead, we wrap the computation in a `Reader` monad and then retrieve the environment when necessary with the `ask` function.

As Solomon writes, and [as shown in the example usage of `Reader` in the documentation](https://hackage.haskell.org/package/mtl-2.2.2/docs/Control-Monad-Reader.html#g:4), we may also use `Reader` to manage variable bindings in an interpreter.

However, it's important to note that this is only useful if you want to interpret a language without global variables or mutation. This makes `Reader` a bad abstraction for modeling variable binding in, say, the Scheme programming language, which crucially has both of these (`define` for global variables and `set!` for mutation). Unfortunately, `Reader` is used to model environments in the [Write You a Scheme Version 2.0 tutorial](https://wespiser.com/writings/wyas/00_overview.html) (WYAS 2), which I think is basically incorrect.

In this post, I will show how `Reader` can be used to model local bindings, and why it's insufficient to represent mutable or global bindings.

## A tiny interpreter

Let's say we want to interpret a language with integer variables and Lisp syntax:

```rkt
(let (x 10)
  x)
```

All our tiny language has are literal expressions (like `10`), variable expressions like the `x` on the second line, and let expressions, like the whole block. When we evaluate the let expression, we traverse the abstract syntax tree (AST), and eventually get the result, which is just the value `10`.

In our interpreter, we represent expressions with a data type called `Expr` [^1]:

```hs
data Expr 
  = Literal Integer
  | Var String
  | Let String Expr Expr -- name, value, body
```

An expression is either a literal integer, a variable which holds an string name, or a let expression, which has a name, a value, and a body.

We can hold our variable bindings in a map from `String` to `Integer`:

```hs
type Bindings = Map String Integer
```

Now we can write a simple evaluation function for our language, `eval`. Our evaluator takes an expression and returns a *computation* representing the variable bindings and the eventual result, an integer. We can implement `eval` like this:

```hs
eval :: Expr -> Reader Bindings Integer
eval (Literal x) = pure x
eval (Var name) = do
  env <- ask
  case Map.lookup name env of
    Just value -> pure value
    Nothing -> error "Undefined Variable"
eval (Let name valueExpr bodyExpr) = do
  env <- ask
  value <- eval valueExpr
  let modified = Map.insert name value env
  local (const modified) (eval bodyExpr)
```

Let's step through each part:
- To evaluate a literal expression, just `pure` the literal value.
- To evaluate a variable expression, we get the current environment with `ask`, then lookup the name in the environment, throwing an error if it's not there.
- To evaluate a let expression, get the current environment; evaluate the value expression; create a new environment with the new binding; and finally, evaluate the body expression in the modified environment. That last part is exactly what `local` does, except it expects a modifying function as its first argument, so we just pass our updated environment as a constant function.

As you can hopefully see, using `Reader` to model local scope is simple to understand and easy to implement. Additionally, the `ReaderT` monad transformer can be used in a transformer stack to evaluate with better error handling and other effects.

## Extending the technique

However, as I alluded to at the beginning of the post, as simple as this model is to understand, it's not really helpful in modeling variable bindings outside of a toy example like this, unless your language only has local scope and no mutation. 

To see this, say we wanted to extend our tiny language with a `define` expression that created a globally scoped binding. The syntax for this would be something like this, in Scheme style:

```rkt
(define x 10)
x
```

which would evaluate to `10`. Say we add a `Define` constructor to our `Expr` type:

```hs
data Expr
 = ...
 | Define String Expr
```

where a `Define` expression holds a name and a value expression. The problem is, when we go to interpret this expression, we don't have a body to evaluate locally like we did with a `Let` expression. The body to evaluate is just implicitly the rest of the program, rather than in a `Let` which makes its scoping explicit. 

We might be able to deal with this problem by peeking ahead in the AST to find a `Define` node, and when we do, evaluating the rest of the tree with the new environment, but this turns out to be pretty inelegant, and it can lead to even more difficulty when handling recursive or nested definitions. This is basically how definitions are evaluated in WYAS 2, which makes what should be a simple feature to implement in my view quite difficult to understand, (see the section ["begin, define & evalBody"](https://wespiser.com/writings/wyas/03_evaluation.html)).

While global definitions might be able to fit into our `Reader` evaluation model, mutation is even more difficult, and as a result WYAS 2 skips adding mutation to the language. When interpreting mutation, as with definitions, we need to deal with the fact that mutation is a side effect, and the mutated state is implicitly in play for the rest of the program. For example, take the following Scheme code:

```rkt
(define x 3)

(let ()
  (set! x 4))

x
```

This should evaluate to `4`, but a naive `Reader` implementation could give `3` since the `set!` mutation occurs in a different scope than the reference to the variable `x`.

## What to do instead

If you want to interpret a language with global definitions and mutation, then, it's best not to use `Reader`. Instead, we can manage a stack of environments, either explicitly or with the `State` monad. With an environment stack,

- To create a local binding, we push a new environment onto the stack, then evaluate the body in this top environment. 
- To lookup a binding, we traverse the stack searching for the first appearance of the name.
- To create a definition, we just modify the top environment on the stack, and continue with evaluation of the rest of the AST.
- To mutate a variable, we traverse the stack searching for the appearance of the name, then return a new stack with the first instance changed.

Concretely, one way to do this would be to define our `Environment` type as a record containing the bindings, and possibly, a parent environment:

```hs
data Environment = Environment {   
    bindings :: Bindings,
    parent :: Maybe Environment
}
```

With this model, by keeping track of the current environment, we can push onto the stack by creating a new environment where the current environment is the parent, and we pop the stack by setting the current environment to the parent. With this, we can add definitions and mutation to our language without dealing with the difficulty of peeking ahead in the tree. Also note that this approach is similar to the implementation of environments in [Nystrom's Crafting Interpreters](https://craftinginterpreters.com/contents.html).

## Conclusion

The usage of `Reader` to implement Scheme environments in WYAS 2 is one of the main reasons why [I have recently considered rewriting the tutorial](https://twitter.com/micah_cantor/status/1478080626523783170?s=20). I think that Haskell is a great choice for implementing an interpreter, but its powerful abstractions must be chosen wisely, and in the case of Scheme evaluation, I think `Reader` is the wrong choice.

That's not to say that the `Reader` trick is entirely useless, but I think it's worth qualifying and understanding its implications. This also all just comes from my experience, and if you are an expert Haskeller who knows how to deal with these issues with using `Reader` for variable bindings, I would love to hear from you.

-----------

### Notes

[^1]: This isn't necessarily how I would structure `Expr` for a real Scheme interpreter, but just an example.