+++
title = "Parsing propositional logic in 33 lines of Racket"
date = 2020-10-11
description = "A guide to implementing a simple parser for propositional logic in Racket."
[taxonomies]
tags = ["racket", "logic"]
[extra]
katex = true
+++

I recently became interested in two seemingly disparate things: Scheme/Racket and symbolic logic, so I decided to have some fun by combining those two into a little project. The marriage is actually a little less weird than it seems, since Racket has very robust tools for lexing and parsing due to its focus on meta-programming. In this post, we'll go over some strategies for implementing a simple parser, with concepts that could easily be extended to parse other things like JSON or even your own programming language.

## What is propositional logic?

Before we go any further, it would be good to introduce what we'll be parsing. [Propositional logic](https://en.wikipedia.org/wiki/Propositional_calculus) is a branch of symbolic logic that defines a grammar and set of rules for stating propositions. With this, we can define an argument such as:

$$
\textnormal{John goes to the park.} \newline
\textnormal{Maria goes to the park.} \newline
\textnormal{Therefore, John and Maria go to the park.} \newline
$$

Or symbolically:

$$
P := \textnormal{John goes to the park} \newline 
Q := \textnormal{Maria goes to the park} \newline
P \ldotp Q \therefore P \land Q
$$

Propositional logic is constrained to sentences that are observational statements like "John goes to the park." There are no variables, and no quantifiers like *all* or *some*.

### The Symbols

With propositional logic informally introduced, let's look at the symbols we have at our disposal, which will need to be worked into our parser:

<table style="margin: 0 auto; border-spacing: 1em 0em">
  <thead>
    <tr>
      <th>Name</th>
      <th>Symbol</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Sentence Letters</td>
      <td> $\text{P, Q, R, ...}$ </td>
    </tr>
    <tr>
      <td> NOT </td>
      <td> $\sim$ </td>
    </tr>
    <tr>
      <td> AND, OR </td>
      <td> $\land , \lor$ </td>
    </tr>
    <tr>
      <td> IF, IFF </td>
      <td> $\rarr , \iff$ </td>
    </tr>
  </tbody>
</table>

Note that $\text{P} \rarr \text{Q}$ is read as "$\text{If P, then Q}$".

## Parsing in Racket

Okay, that's enough theory, let's get to coding. The main libraries at our disposal are found in the [parser-tools](https://docs.racket-lang.org/parser-tools/) package, but the documentation for these tools was confusing to me, and setting up a simple parser was not very ergonomic. Fortunately, there is another way of building a parser: the Brag package.

[Brag](https://docs.racket-lang.org/brag/) (Better Racket AST Generator) is a package that allows us to define a grammar in standard BNF form, then easily lex and parse that grammar. Simply install with `raco pkg install brag` so we can use it.

### Step One: Defining our grammar

Let's create a new file in our directory called `grammar.rkt` and put `#lang brag` at the top. This is the file we will use to define our BNF grammar. For propositional logic, that grammar can be represented elegantly, like this:

``` racket
#lang brag
sentence: ATOMIC | complex
complex: LPAR sentence RPAR | sentence connective sentence | NOT sentence
connective: AND | OR | IF | IFF
```
In Brag, any string in all caps is interpreted as a token to be included in our lexer, while those not in caps are types of expressions that we must define elsewhere in the grammar. We define the tokens ATOMIC, NOT, AND, OR, IF, IFF as analogues for the symbols we discussed earlier. We also have the tokens LPAR and RPAR to represent parentheses.

Here, we are successively breaking down the syntax of the language into its constituent parts. A `sentence` can be either `ATOMIC` (e.g $P$ or $Q$) or `complex`. A `complex` expression is any sentence within parentheses, or two sentences joined by a connective, or a negation of a sentence. Finally, a connective can be AND, OR, IF, or IFF.

### Step Two: Creating the lexer

Let's create another racket file in the same directory called `parser.rkt`. We also need to import `brag/support`, our grammar from `grammar.rkt` and the package `br-parser-tools/lex`, which is a fork of the default racket parser tools used by brag. To install this, simply `raco pkg install br-parser-tools`. You should now have this:

```racket
#lang racket
(require brag/support)
(require br-parser-tools/lex)
(require "grammar.rkt")
```

Now we can now define our tokenize function, which will take in an input string, and output a list of tokens. It will have the form of something like this:

```racket
(define (tokenize ip)
    (define lexer
      	(lexer-src-pos
            ;; define token rules here!
       		[(eof) (void)]))
    (define (next-token) (lexer ip))
    next-token)
```

All we are doing here is setting up the imported function `lexer-src-pos` which will take a list of rules for matching tokens and return a function that lexes them. Then, by calling `next-token` we run this lexer function recursively on the input port until we reach the special token `eof`, at which point we return void.

We can now add in the rules for the tokens we want to lex for:

```racket
(define (tokenize ip)
    (define lexer
      	(lexer-src-pos
          [(char-range #\P #\Z) (token 'ATOMIC lexeme)] ; matches characters between P and Z
          ["^" (token 'AND lexeme)] ; matches "^" as logical AND token
          ["v" (token 'OR lexeme)] ; matches "v" as logical OR token
          ["~" (token 'NOT lexeme)] ; matches "~" as NOT token
          ["->" (token 'IF lexeme)] ; matches "->" as IF token
          ["<->" (token 'IFF lexeme)] ; matches "<->" as IFF token
          ["(" (token 'LPAR lexeme)] ; matches "(" as left parentheses
          [")" (token 'RPAR lexeme)] ; matches ")" as right parentheses
          [whitespace (token 'WHITESPACE lexeme #:skip? #t)] ; skips whitespace
          [(eof) (void)])) ; return void at end of file, this stops the parser
    (define (next-token) (lexer ip)) ; moves lexer to find the next token
    next-token)
```

Notice how each of these rules are enclosed in brackets, where the first expression is the pattern to lex, and the second expression is what to return with that pattern. You can return anything you want (or add side effects) but we will return the `token` struct built into Brag, since this will integrate well with the package's parsing capabilities.

The `token` struct takes two inputs: the name (which we defined in our grammar!), and the lexeme, which is the string returned from the pattern we just lexed. So, when we add `racket› ["^" (token 'AND lexeme)]`, we are lexing for the string `"^"` under the syntactic rules for the token `AND` defined in our grammar file.

Also note that the lex package comes with some functions useful for matching tokens like `char-range` and `whitespace` which we use here too.

### Step Three: Parse and Display

We now have a function that can tokenize any input string, so let's parse those tokens now, which is really as simple as:

```racket
(define stx
	(parse (tokenize (open-input-string "P^Q")))) ; produces syntax struct
(syntax->datum stx) ; converts syntax to the datum it contains
```
Parser complete! And as promised, the grammar and parser files amount to just **33 lines**.

This produces a list of tokens, which for the input `"P^Q"` is 

``` racket
'(sentence 
    (complex 
      (sentence "P") 
      (connective "^") 
      (sentence "Q")))
```
This is exactly the data structure we want. The entire string is a sentence, with a complex sentence within it, containing the tokens P, ^, and Q.

Let's try it on something a bit more complicated: `"(P->Q) <-> ~R"`:

```racket
'(sentence 
    (complex (sentence (complex 
      "(" (sentence (complex (sentence "P") (connective "->") (sentence "Q"))) ")"))
      (connective "<->") 
      (sentence (complex "~" (sentence "R")))))
```
Again, this nested structure is exactly what we want.

## Where to go from here?

We can now parse any symbolic sentence in propositional logic. Parsing is a great first step, but on it's own, it doesn't do very much. With the data structures we produce though, we could create programs that visualize the logical tree, derive theorems and arguments with a solver, among others. The same strategies we used here could also be applied to parsing JSON or other structured languages.

And, for a more in-depth look at Brag, make sure to read [the docs](https://docs.racket-lang.org/brag/).

Questions or comments? [Email me](mailto:micahcantor01@gmail.com).