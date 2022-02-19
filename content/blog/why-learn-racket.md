+++
title = "Why learn Racket? A student's perspective"
date = 2022-01-25
description = "A few thoughts on why I think you should learn Racket."
[taxonomies]
tags = ["racket", "programming-languages"]
+++

I'm a student at Grinnell College, a small liberal arts school in Iowa. Here, the computer science department uses Racket in its intro course, which focuses on functional programming and is aimed at all students (majors, non-majors, and those with and without prior programming experience.) After taking the course myself last fall, this semester I am working as a student mentor for the course, where I help students during in-class labs and hold weekly review sessions.

Grinnell is far from alone in its choice to use Racket or Scheme in its introductory CS course. Perhaps the most well known Scheme course is MIT's 6.001, which began in 1980 and taught computer science using the book [Structure and Interpretation of Computer Programs](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book.html). MIT replaced their Scheme curriculum with a Python course in 2008, but other universities like Brown, Northeastern, and Indiana maintain Scheme or Racket in their first courses. Still, perhaps following MIT's lead, the use of Racket for an intro course is still relatively rare among CS undergraduate programs.

Given that Racket isn't a widely popular language, several students at Grinnell, often those who enter the class with prior experience in more popular languages like Python or Java, ask why it's chosen for Grinnell's first course. I've also heard frustrated students wonder why they're wasting their time on a "useless" language, when they could learn a language that will help them get a job instead. Our professor gives an answer to those who ask, and there are other resources I've seen such as Matthew Butterick's excellent piece, [Why Racket? Why Lisp?](https://beautifulracket.com/appendix/why-racket-why-lisp.html), but here I'd like to share my own opinion on why I think learning Racket is worthwhile. 

I think Racket is a great first language to learn, but in this post, I'd like to concentrate on programmers, college-bound or otherwise, who have some experience in a mainstream language like Python, JavaScript, or Java, but are hesitant or unsure of the value in learning a functional programming language like Racket. 

## Mental models of computation

Racket, and Lisps more generally, are often touted as languages with the simplest syntax. In Racket, every form is an expression, which is either an atomic value like a string or number, or a list beginning with a procedure or a special form. This syntax stands in contrast to a language like Python, which has strict rules for which forms may be used as expressions and which must be statements.

I think this advantage is sometimes overstated by Lisp enthusiasts. Although Python's syntax may not be as simple, it does have its own kind of elegance once a user learns its syntactic rules. Where Lisp-style syntax does have an advantage, however, is in the power of its mental evaluation model. When evaluating Racket code, we always evaluate the innermost parentheses first, replacing procedure calls with their appropriate body, except for a few special forms.

For example, take the following Racket expression to square every number in a list

```rkt
(map sqr '(2 4 8))
```

To evaluate this, we replace `map` with the body of its definition, substituting the arguments in place, and then continue with normal evaluation. We can trace an abbreviated version like this:

```rkt
(map sqr '(2 4 8))
-> (if (null? '(2 4 8))
        null
        (cons (sqr (car '(2 4 8))) (map sqr (cdr '(2 4 8)))))
-> (cons 4 (map sqr '(4 8)))
-> ...
-> (cons 4 (cons 16 (cons 64 null)))
-> '(4 16 64)
```

Following the trace, we an see that there is nothing magical about `map`, it's just a procedure that evaluates like any other. Compare that now to the equivalent list comprehension in Python:

```py
[x*x for x in [2, 4, 8]]
```

How can we mentally evaluate this? For that we need remember the specific "magical" syntax and semantics for evaluating Python's for-comprehension expressions, which look similar yet act completely differently than for-loop statements.

Racket has special forms too like `let` and `if` for which evaluation strategies must be learned, but these are generally more limited and intuitive than the syntax found in Python or Java.

Furthermore, what's our mental model for evaluating this Java code?

```java
public class HelloWorld {
  public static void main(String[] args) {
    System.out.println("Hello World!");
  }
}
```

In just a few simple lines of Java's canonical "hello world" example, we've made several declarations with special rules for evaluation (`public`, `class`, `static`, the dot operator, etc.) These all must be understood before a Java programmer can know much of anything about what their program is actually doing. 

Java was the first programming language that I learned, so I know from first-hand experience that the result is that beginners to the language usually just accept that they can't fully understand how even a simple program like this is evaluated. They're forced to accept that learning to program involves memorizing seemingly arbitrary rules for structuring their code. Beginner Racket programmers face no such imposition, as its evaluation is much more intuitive, which I see as a clear advantage in its use as a teaching language.

Even if you are an experienced Python or Java programmer that knows exactly how every part of the language can be evaluated, I still think that exposure to Racket's evaluation model is worthwhile. Modeling how Racket code is transformed builds your intuition as a programmer for how different pieces of code decompose and fit together in a way that users of other languages usually don't directly consider. I think that strengthening these fundamentals builds intuition for quickly grasping evaluation in other languages.

## Less is more

Learning Racket often involves working with a more limited set of tools than the ones offered in other languages. Loops and mutation, for instance, which are core features in other languages, are usually off-limits to beginners in Racket 

While there are genuine aesthetic advantages to a language built with a small core, namely in how it can be concisely defined and implemented, I think it's important not to overstate these in comparison to the material advantages of a small language as well. 

The focus on minimalism in Racket initially forces even experienced programmers to step out of their comfort zone in how they structure their code, and ultimately, how they solve problems. For experienced programmers, I think there is great value in learning fundamentally different ways to approach a problem. This kind of experience pushes the bounds of one's knowledge and adds to a "bag of tricks" that allow one to solve more difficult or novel problems.

## Thinking recursively

Let's talk about recursion, because I think that learning to use recursion as a general-purpose control structure is one of the most important consequences of gaining fluency in Racket.

When programmers in other languages are introduced to recursion, it's usually to display an alternative and niche solution to some problems. Canonical examples like calculating a factorial or the nth Fibonacci number are given, even though these toy examples seem useless to programming at large. Additionally, programmers are often warned not to use recursion at all because of stack overflow concerns.

Recursion, however, is a much more powerful tool than these beginner tutorials let on. Indeed, without loops and mutation, recursion is the main tool employed in Racket to manage repetition and state, and learning to use it as such is one of the most challenging aspects of learning the language for beginner and experienced programmers alike.

The key to programming with recursion isn't that it can be used to implement recursive mathematical definitions, but instead that it's a general tool for breaking down a single difficult problem into several easier ones. This decomposition into sub-problems is, in fact, what allows recursion to work at all.

Let's look at an example, comparing an iterative and recursive design of a `sum` procedure. Here is a standard iterative approach in Python:

```py
def sum(lst):
  total = 0
  for x in lst:
    total += x
  return total
```

This approach follows directly from how most people would answer the question *"how do you sum a list of numbers?"* Well, start with zero, then add each value to the total until we reach the end. 

Implementing `sum` recursively, however, requires a different mindset. Instead, our strategy will be to ask two questions: First, *"what's the sum of an empty list?"* Okay, obviously zero. Next, *"if we know the sum of all but the first element in a list, what's the total sum?"* Here we can find this by adding the value of the first element to the sum of the rest.

Putting this strategy into Racket, we would get this:

```rkt
(define (sum lst)
  (if (null? lst)
      0
      (+ (car lst) (sum (cdr lst)))))
```

This line of thinking is certainly not intuitive to someone unfamiliar with recursive design. However, it importantly forces us to split our design into two sub-problems: first solving the base case, and then breaking down the recursive step into what can be handled by the recursive call, and what we must do at each stage.

Recognizing the value to thinking recursively can be difficult, since it's not obvious why adopting an inherently less intuitive strategy to problem solving is helpful. But the beauty of recursion lies how it allows us to solve difficult problems and manage state with inherently less powerful tools (no mutation or loops), and how that simplicity rewards experienced programmers with the ability to quickly break down difficult problems.

Recursion isn't a silver-bullet, but it's a fundamental tool that's worth understanding, even if functional programmers usually prefer to use more specific and safer tools built on top of recursion like higher-order procedures or comprehensions.

## Racket and recursion

Even if all this waxing about recursion is true, one may still wonder why to choose Racket when recursion is available in every other programming language as well. The answer is that Racket, like most other functional languages, is designed to make recursive design more ergonomic and natural. 

First, every Racket procedure implicit returns the final expression, so there is no possibility of writing a foot-gun recursive procedure that doesn't return a value. Additionally, core forms like are designed to discourage imperative programming. For example, an `if` expression must include truthy and false branches, so it's more difficult for one to forget to include a base case.

Perhaps most important, however, is the core placement of the list data structure. Lists are an inherently recursive data structure, as they're either `null` or `cons` of some value and another list. Recursive design over `cons` lists is natural, so their central placement in Racket makes using recursion a natural choice. 

Compare that to using recursion in a language like Python. There the primary list data structure is an indexed dynamic array. Although one can translate the same recursive algorithms one would write with `cons` and `cdr` in Racket to Python, doing so would not only be unnatural, but also goes against the ethos of Python. And really, the same would go for a translation to other popular imperative languages.

Racket is advantageous for recursive design from the perspective of performance, rather than just pedagogy, too. Importantly, Racket guarantees [tail call optimization](https://en.wikipedia.org/wiki/Tail_call), which means that recursive functions designed to exploit tail recursion won't explode the memory on the stack as they would in imperative languages like Python and Java.

## Languages are less important than you might think

Now that I have extolled the virtues of learning Racket, I want to add that the choice of a programming language is less important than you might think. One of the main areas of frustration I see from fellow students essentially boils down to the following question: *"Why should I learn Racket when I could spend that time learning a language that people actually use?"* On the surface, this question makes some sense. Time and effort are scarce resources that should be allocated wisely, but I think asking this question misses a vital point: that *programming* is much more difficult to learn than a *programming language*. 

I don't want to trivialize the effort needed to learn a new language, especially one like Racket that looks so different than popular C-like languages. But the syntax and specific features of a given language still only make up a small part of the knowledge required to master programming in general, in that language or any other. Indeed, experienced programmers can learn a new language much faster than novices, since they already know most of the concepts that underlie the new language, so they can focus on picking up the syntax or learning language-specific features instead.

When a Racket programmer picks up JavaScript for the first time, they already know how to use anonymous functions, closures and higher-order functions, so they only need to map their existing knowledge onto JavaScript's syntax and focus on its distinct features. This advantage only increases over time, since the more languages that one learns, the more knowledge bases one has to draw on when picking up a new language.

So yes, you will have to invest time into learning the specifics of Racket, but the reward for that effort is that most of the knowledge obtained will pay dividends when you do move on to other languages. And because the core of Racket is so small and so expressive, it allows users to quickly dive into those more difficult yet ultimately more valuable aspects of programming, rather than getting bogged down in the heavy syntax and quirkiness found in languages like Java.

## Functional programming isn't useless in practice

The benefits of functional programming are often extolled from an intellectual point-of-view. The story goes that functional programming's focus on purity and statelessness will carry over to produce better results in imperative programming languages that de-emphasize those aspects. I agree with this sentiment, since I feel as though programming in a near-functional style is a useful default in any language, while still reaching for mutation and state when they're needed.

That being said, even if the large majority of programmers and engineers in industry use non-functional languages, to say that functional programming by itself is entirely useless in practice is inaccurate.

This summer, for instance, I will be working as a software engineer intern for a company that uses Clojure in its back end, an opportunity that I only could have gotten because of my experience with Racket. I say this not to brag, but to point out that if you want to work with functional programming professionally, you may be able to get by even if Racket is the only language on your resume. 

The functional programming community is certainly smaller than others in software engineering, but that's not entirely a bad thing. In my (albeit limited) experience, smaller opportunities can often be more accessible and more rewarding than flagship job or internship programmers that attract a massive amount of applications. In the end, every applicant needs something that makes them stand out, and if functional programming is their passion, that could be it.

Additionally, and perhaps more importantly, even if the functional programming community within software engineering is relatively small, the importance of functional tools and techniques in almost every modern language has recently grown considerably. 

In the last few years, bastions of object-oriented programming like [Java](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html) and [C#](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions) have added support for lambda expressions, and [Python introduced structural pattern matching](https://www.python.org/dev/peps/pep-0636/) -- both important tools lifted from functional languages. 

Not only that, but [React](https://reactjs.org/), and its functional approach to managing state with hooks, has become the de-facto JavaScript UI library in the frontend web development community. Even if functional programming languages lag behind imperative or object-oriented ones in adoption, their techniques and features have increasingly bled over to the point that they're now unavoidable.

## Final Thoughts

Racket is above all, an excellent programming language, and in my view, a valuable learning tool for novice and experienced programmers alike. That's why it frustrates me when my peers complain about the language for what I see as irrational reasons. Not everyone needs to love or prefer functional programming in Racket, but I hope this post at least helps some of those people appreciate its value.