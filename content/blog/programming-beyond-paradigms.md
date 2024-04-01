+++
title = "Programming beyond paradigms"
date = 2024-03-31
description = "Why the next paradigm in programming is no paradigm at all."
[taxonomies]
tags = ["programming-languages"]
[extra]
show_toc = false
+++

Since the invention of high-level programming languages, programming has been dominated by paradigms. 
From imperative to object-oriented to functional, paradigms shape much of the modern discourse around programming languages.

Paradigms are often used to categorize the many different languages, styles, and eras of programming.
Many universities also structure their curricula around paradigms, offering courses in different styles of progamming.
And of course paradigms are an endless topic of debate among programmers: 
countless blog posts have been written on why functional is superior to object-oriented programming, and vice-versa.

This leads to a natural question: *what really is a programming paradigm?*
The most common understanding is that a paradigm is a set of features in a progamming language that determine its control flow or type system.
For instance, this definition can be applied to three prototypical languages created in the 20th century:

- C is an imperative language; its programs are composed of statements and procedures that are executed top-to-bottom.
- Java is an object-oriented language; its programs are composed of objects that interact through dynamic dispatch.
- Haskell is a functional language; its programs are evaluated by the composition and application of pure functions. 

Programming languages of the past were limited in the number and breadth of features they could offer. 
For Java and Haskell, the language designers chose a set of features that they felt would produce the best programs.
The creators of C did the same, but were also limited by the feasibility of a feature's implementation.

But modern programming languages are far less limited in the scope of features they choose to include and their capacity to do so.
This has led many languages to adopt a new and somewhat confounding adjective: *multi-paradigm*.

Indeed, almost all modern languages are best described as multi-paradigm, which deflates the significance of the term. 
The latest versions of Python, JavaScript, and even Java all offer the features needed for functional programming alongside imperative and object-oriented progamming. The same could be said for C++, C#, OCaml, and more.

Due to the relative feature-set equality between languages, I believe that programming paradigms are now best understood as a *style* of programming, rather than as an exclusive set of features. Since almost all languages are multi-paradigm, we can't categorize an entire language under a single paradigm. But we can still compare how particular programs, or parts of programs, use one paradigm or another across languages.

General purpose languages of the 21st century don't define themselves by a single paradigm anymore, nor do they use the term multi-paradigm.
The unifying aspect of new languages such as [Rust](https://www.rust-lang.org/), [Nim](https://nim-lang.org/), and [Gleam](https://gleam.run/) is that they were designed from the beginning to be **beyond paradigms**.

These post-paradigm languages draw on the best features of past languages and refuse to limit the programmer to a single style. 
This puts more power and creativity in the hands of the programmer rather than the language designer. 
Programming styles in these languages are meant to be mismatched and combined, rather than adhered to under a single dogmatic paradigm.

Programming languages of the future will continue to prioritize the inclusion of the best individual features in their language or tooling, regardless of paradigm. I think there will continue to be a great diversity between different languages, but the best aspects of each programming paradigm refuse to be siloed.



