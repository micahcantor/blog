+++
title = "Compiling a subset of JavaScript to ARM assembly in Haskell"
date = 2022-05-29
description = "A toy compiler for a subset of JavaScript to ARM assembly, using Haskell."
[taxonomies]
tags = ["haskell", "programming-languages"]
+++

I recently got a copy of the book [*Compiling to Assembly from Scratch*](https://keleshev.com/compiling-to-assembly-from-scratch/) by Vladamir Keleshev, which details how to write a compiler for a subset of JavaScript to 32-bit ARM assembly code. The choice to use ARM assembly is mainly for its simplicity in comparison to x86. 

Keleshev elects to use TypeScript to write the compiler, which, as he explains in the preface, is largely a compromise for its familiar C-like syntax while still providing many functional programming features desireable for writing a compiler. However, I chose to write my version of the compiler in Haskell as it's my favorite language, and the focus on functional programming from Keleshev makes it a natural choice for the translation.

Overall, I had a lot of fun writing this compiler as I got to learn more about the nitty-gritty low-level of how code is executed while getting more practice with Haskell. In this post, I won't cover every detail of the design of the compiler, but I'll try to hit on what I found to be the most important or interesting aspects of the code. 

## The Abstract Syntax Tree

The first step in writing our compiler is to define a data structure that abstractly represents the syntax of our source language, JavaScript. Once we convert the program source text to this internal representation (through parsing), our job will become to recursively traverse this tree.

We represent the abstract syntax tree (AST) in Haskell as a large sum type that looks like this:

```hs
data Expr
  = Number Integer
  | Identifier Text
  | If Expr Expr Expr -- condition, consequent, alternate
  | Add Expr Expr -- left, right
  ... -- other expression types here
  deriving (Eq, Show)
```

For each type of expression we want to represent, like number literals, identifiers or if expressions, we add a type constructor for that expression to `Expr`. With this data structure ready, we can now move on to parsing.

## Parsing

The parser is the first step in our compiler pipeline. In the book, Keleshev chooses to write and use a small parser combinator library for his parser. Those familiar with Haskell will know this actually happens to be a real specialty of the language and the community, so translating this portion of the compiler was particularly natural.

I chose to use the [megaparsec](https://hackage.haskell.org/package/megaparsec) parser combinator library rather than writing my own as Keleshev does (though that would have been a fun exercise in itself). Parser combinators are small primitive functions that compose together in order to write a high-level parser that closely aligns with the prescribed grammar for the language.

The main difference between the Haskell and TypeScript code in this section is Haskell's support for monadic programming, which Keleshev humbly tries to imitate in TypeScript. An example should illuminate the difference here.

One part of the parser is for if statements. In TypeScript, the parser looks like this:

```ts
let ifStatement: Parser<AST> =
  IF.and(LEFT_PAREN).and(expression).bind((conditional) =>
    RIGHT_PAREN.and(statement).bind((consequence) =>
      ELSE.and(statement).bind((alternative) =>
        constant(new If(conditional, consequence, alternative)))));
```

This parses a sequence of tokens: the `if` token, then a left paren, then some expression which we bind as the conditional, then a right paren, then the consequence and alternative statements. We translate this code directly to Haskell using `do`-notation, which implicitly adds the `and` and `bind` higher-order functions used by Keleshev:

```hs
ifStatement :: Parser Expr
ifStatement = label "if statement" $ do
  _ <- ifToken
  condition <- parenthesized expr
  consequent <- statement
  _ <- elseToken
  alternate <- statement
  pure (If condition consequent alternate)
```

Here, since we don't need the parsed text from `ifToken` or `elseToken`, I just bind them to the empty identifier, `_`. This parser also uses a helper function called `parenthesized`, which takes a parser and outputs the same parser that matches only between two parentheses, so 

```hs
parenthesized :: Parser a -> Parser a
parenthesized = between leftParen rightParen
```

Personally, I find the `do`-notation to be very helpful in increasing the readability of code like `ifStatement` which is written using monads, since it allows us to flatten the code and make it clear the order in which we are binding intermediate results.

There are other benefits to using an established parser combinator library like `megaparsec` rather than rolling our own. For example, one small part of the parser is parsing the list of arguments in a function call. For example, in a function call like

```js
foo(1, 2 + 2, a);
```

we want a parser consumes this comma-separated list of expressions and returns `Parser [Expr]`, which is a list of expressions wrapped in the carrier `Parser` type. Specifically we should parse the arguments as the list

```hs
[Number 1, Add (Number 2) (Number 2), Identifier "a"]
```

Keleshev's TypeScript to parse this kind of expression looks like this:

```typescript
let args: Parser<Array<AST>> =
  expression.bind((arg) =>
    zeroOrMore(COMMA.and(expression)).bind((args) =>
      constant([arg, ...args])))
  .or(constant([]))
```
Essentially, this parser either finds one expression followed by zero or more comma separated expressions, or nothing, in which we return an empty list. 

In Haskell, this job is easier, since `megaparsec` provides a useful `sepBy` combinator that does this for us. So we could instead write

```hs
args :: Parser [Expr]
args = expr `sepBy` comma
```

where `expr` parses any expression and `comma` parses a comma token with optional whitespace. But this parser is simple enough that in my code I keep this inline in the parser for function calls.

Once you get used to the style and idioms of parser combinators, the job of converting from a grammar to a parser for that grammar becomes fairly mechanical, yet `megaparsec` still gives great error messages and performance. 

## Code Generation Framework

With the parser set, we can now take a basic JavaScript program and translate it into the corresponding AST represented by our `Expr` type. Now we need to process that `Expr` and produce a text file of assembly code which we can then assemble and execute.

This suggests that our `emit` function to produce the assembly source should be

```hs
emit :: Expr -> Text
```

But, while we do want to output some text, we need to track some effects at the same time, namely compile time errors in the code and the state of variable bindings and other values. 

Therefore, we take a traditional approach to structuring a Haskell program which is to build a stack of monads in which to perform these effects, and output that from our `emit` function instead. First, though, we need some data types that will be tracked in these effects.

We have a simple `CompilerError` data type that enumerates the possible compiler errors and an associated error message:

```hs
data CompilerError
  = Unsupported Text
  | UndefinedVariable Text
  deriving (Eq, Show)
```

Right now we have only two compiler errors, `Unsupported` for unimplemented features and `UndefinedVariable` for, well, undefined variables.

Then we have a `Context` data type that tracks three pieces of information we need throughout the code generation:

```hs
data Context = Context { 
  locals :: Map Text Int, 
  labelCounter :: Int, 
  nextLocalOffset :: Int
}
```

First is `locals`, a map of our variable bindings to their offset location in the stack. Second is `labelCounter`, which is a counter we need to generate unique labels that don't clash with each other, and lastly `nextLocalOffset` tracks where in the stack the last variable binding is located.

Finally, we want to progressively build up `Text` with the assembly source, so the most efficient way is to do this is to use the `Builder` type from the `text` package. So we have a simple type alias:

```hs
type AsmSrc = Builder
```

With all of that in place, we can finally build our monad transformer stack:


```hs
newtype Emit a = Emit {unEmit :: StateT Context (WriterT AsmSrc (Except CompilerError)) a}
  deriving (Functor, Applicative, Monad, MonadState Context, MonadError CompilerError, MonadWriter AsmSrc)
```

This is a computation that tracks three effects: the state of the code generation with `StateT Context`, any compiler errors with `Except CompilerError`, and the generated text builder, in `WriterT AsmSrc`. By using the monad transformers from `mtl`, we can access all of these effects in one data type, which we call `Emit`.

Then we need some way of pulling out either the compiler error or the generated code, so we define the following function do so:

```hs
execEmit :: Context -> Emit a -> Either CompilerError AsmSrc
execEmit state emit = runExcept (execWriterT (execStateT (unEmit emit) state))
```

which unwraps each monad from the inside-out.

With that framework set up, we can finally get around to defining our `emit` function, which now has the type

```hs
emit :: Expr -> Emit ()
```

meaning that it takes in an expression, and returns an `Emit ()` computation. There is no value to wrap in that computation, since this procedure is all "side-effects," namely tracking state, throwing errors, and building up the generated assembly.

We add a short helper procedure to add a line to our assemply source, which we will use often:

```hs
addLine :: Text -> Emit ()
addLine ln = tell (Builder.fromText (ln <> "\n"))
```

This function uses `tell` to append the given text with a newline to the `Builder` that lives in our `WriterT` monad. With this we can write 

```hs
addLine ".global main"
```

to add that line of text to our assembly.

## ARM-32 Assembly Code

Now that we have this monadic framework set up, how do we actually generate aeembly code? The general idea is to visit each node in our AST and emit some assembly code based on the type of that node. Specifically, we generally put intermediate values in the program into the "argument registers", which in ARM assembly are the first four registers, `r0` through `r3`. Let's look at some examples:

### Booleans

To compile booleans, we place the number 1 into the first register, `r0`, if the value is true, and otherwise place the number 0 into `r0`. In our `emit` procedure, that looks like this:

```hs
emit :: Expr -> Emit ()
emit expr = case expr of
  Boolean val ->
    if val then
      addLine "  mov r0, #1"
    else
      addLine "  mov r0, #0"
```

This means that our representation for `true` and `false` is essentially identical to just using 1 and 0, we aren't distinguishing those types. Also note that for readability of the generated assembly, we are manually moving over instruction lines by two spaces at the start of each text literal. Now let's look at numbers.

### Numbers

We only deal with integers in this toy compiler, so to compile them, we just put the value of integer into `r0`.

```hs
-- in emit :: Expr -> Emit ()
Number val ->
  addLine ("  ldr r0, =" <> toText val)
```

We use the pseduo-instruction `ldr` (short for load register) here instead of `mov` in case that the number does not fit into the relatively small value of the immeditate. With that literal assumed to be in `r0`, we can access it when processing another AST node such as `Add`. 

### Infix Operators

Compiling an infix expression like `3 + 4` is a bit more complicated. Our first thought would be to put the left value in `r0` and the right value in `r1`, then add them, but that would fail if the left expression was more complicated and could not fit entirely in `r0`, such as some nested expression like `(5 + 3) + 2`. In that case it would clobber the right expression stored in `r1`.

Instead, we have to make use of the stack. Our strategy will be to emit the left expression, push it onto the stack, then emit the right expression, and pull the left back into `r1`. Then even if we have a nested expression they won't overwrite each other in memory.

There is an extra complication to using the stack, however. When the call stack interacts with external interfaces, such as calling a `libc` function (which we will use to provide basic functionality such as printing characters), the stack pointer (the address of the start of the stack) must be aligned at the 8-byte boundary. For simplicity, Keleshev elects to always maintain this invariant when utilizing the stack. This means if we want to push a single word onto the stack, we must also push some dummy value, such as the `ip` register, as well.

In the end, our assembly code for `Add` looks like this:

```hs
Add left right -> do
  emit left
  addLine "  push {r0, ip}"   -- push left (r0) onto stack with alignment 
  emit right
  addLine "  pop {r1, ip}"    -- restore left result from stack into r1
  addLine "  add r0, r1, r0"  -- add r1 and r0, store result in r0.
```

In fact, the code to compile other infix operations like subtraction, multiplication, and equality comparison would be identical except for the last line. Therefore, we can refactor this to write a more general `emitInfix` function, which takes some `Emit` action to place after the standard code for working with the left and right expressions:

```hs
emitInfix :: Expr -> Expr -> Emit () -> Emit ()
emitInfix left right action = do
  emit left
  addLine "  push {r0, ip}"  -- push left (r0) onto stack with alignment 
  emit right
  addLine "  pop {r1, ip}"   -- restore left result from stack into r1
  action                     -- perform action on r0 and r1
```

Then we can rewrite the `emit` case for `Add` simply as

```hs
Add left right -> emitInfix left right $
  addLine "  add r0, r1, r0"
```

The equality infix operator `==` works the same way, except instead of adding the two values, we compare them with the `cmp` instruction, returning 1 if they are equal and 0 otherwise. It looks like this:

```hs
Equal left right -> emitInfix left right $ do
  addLine "  cmp r1, r0"    -- compare left to right
  addLine "  moveq r0, #1"  -- if equal, store 1 in r0
  addLine "  movne r0, #0"  -- otherwise store 0 in r0
```

The instruction `moveq` only executes if the result of `cmp r1, r0` was that they were equal, and the opposite for `movne`.

### If Statements

We can make use of the same kind of conditional logic we saw in the implemenation of the equality operator to compile if statements.

Here, we also need the concept of labels, which are simply named blocks of assembly that we can branch to from some other part of the code. Essentially, to compile an if statement, we create a label with the falsey-branch and a label for the code that runs after the if statement. Then we emit the condition expression and check if it is truthy. If it is, we continue onto the consequent block, and otherwise branch to the alternate block. Afterwards, we always branch to the rest of the code.

We can name the labels anything, but we need to be careful because we need them to be unique so that if there are two if statements in a program, the labels do not conflict with each other. Therefore we carry around a counter in our `Context` that we use to create a unique label, with this `makeLabel` function:

```hs
makeLabel :: Emit Text
makeLabel = do
  incrementLabelCounter
  counter <- gets labelCounter
  pure (".L" <> toText counter) -- prefix "L" is standard for code generation 

incrementLabelCounter :: Emit ()
incrementLabelCounter = modify (\s -> s {labelCounter = labelCounter s + 1}) 
```

The helper function `incrementLabelCounter` simply adds one to the label counter and adjusts the state in the `Emit` monad. Now here is the full code for the `If` case of the `emit` function, where the `b` instruction is short for "branch":

```hs
If condition consequent alternate -> do
  ifFalseLabel <- makeLabel -- unique label for falsey branch
  endIfLabel <- makeLabel   -- unique label for after if is evaluated
  emit condition            -- emit condition code
  addLine "  cmp r0, #0"    -- check if condition is false
  addLine ("  beq " <> ifFalseLabel) -- if so, branch to ifFalse label
  emit consequent           -- emit consequent, executed only if we do not branch to ifFalse 
  addLine ("  b " <> endIfLabel) -- branch to endIf label either way
  addLine (ifFalseLabel <> ":")  -- define ifFalse label
  emit alternate                 -- ifFalse label contains alternate code
  addLine (endIfLabel <> ":")    -- define endIf label with whatever comes next
```

As described above, we create two unique labels, then arrange the code as necessary within these new labels.

### Variable Bindings

Next up in our whirlwind compiler tour is variable bindings. In our `Context` we store a `Map` called `locals` where the keys are the names of the variables and the values are their offset from the current frame pointer `fp`. Our plan is to store local variables in the current stack frame, and when we need to retrieve one, we lookup its name in `locals` and reference that memory location.

The only complication is that again, we store all the variables at two-word, 8-byte boundaries in the stack. To do this, we keep track of the `nextLocalOffset`, which is a running count of where to put the next local variable. In the end our `emit` case for `Var` looks like this:

```hs
Var name value -> do
  emit value
  addLine "  push {r0, ip}"
  Context{locals, nextLocalOffset} <- get
  setLocals (Map.insert name (nextLocalOffset - 4) locals)
  setNextLocalOffset (nextLocalOffset - 8)
```

Here we just emit the value into `r0`, push it onto the stack with alignment, then update our `locals` map and set the next local offset down 8 bytes. This is because the stack actually grows down in memory from the stack pointer. 

### Identifiers

Now we need to implement the reverse, pulling the value for a variable identifier. To do this, we implement a higher order procedure called `withVarLookup`. This function takes the name of a variable and a function that takes its offset as an input for some `Emit` action. After we look up the offset of the parameter, we pass it to this given function, and if the name does not exist in `locals` we throw an error:

```hs
withVarLookup :: Text -> (Int -> Emit ()) -> Emit ()
withVarLookup name withOffset = do
  env <- gets locals
  case Map.lookup name env of
    Just offset -> 
      withOffset offset
    Nothing ->
      throwError (UndefinedVariable name)
```

With this we can implement the `emit` case for `Identifier`, which is an AST node which holds the name of a variable:

```hs
Identifier name ->
  withVarLookup name $ \offset ->
    addLine ("  ldr r0, [fp, #" <> toText offset <> "]")
```

So to compile an identifier, we look up the name of the variable, then load `r0` with the value of that variable, which is offset from `fp`. That bracket syntax allows us to reference a memory address, so for example `[fp, #-4]` would reference the memory location at the frame pointer minus 4 bytes. This is exactly what we want, except pluggin in the appropriate offset.

### Function Calls

Even though we haven't yet implemented function definitions, we are going to first implement function calls. A function call looks like `foo(1, 2)` in JavaScript and is parsed into a `Call` node by our parser which has two components: the name of the callee and the list of arguments. 

In ARM assembly, the convention is that the first four registers are for function arguments. If we have more than four arguments, they must be put on the stack, and for simplicity this difficulty is avoided in the baseline compiler in Keleshev's book. 

In our compiler, we must still use the stack, however, since if we try and move the arguments directly into the registers they could clobber each other. So we will allocate memory on the stack, then emit each argument and store it in the appropriate spot. Then we pop the values back into the argument registers and call the `bl` instruction to "branch and link" to the name of the callee. In the end it looks like this:

```hs
Call callee args -> do
  let count = length args
  if count == 0 then
    addLine ("  bl " <> callee) -- branch and link to function name
  else if count == 1 then do
    emit (head args) -- if one arg, just emit it
    addLine ("  bl " <> callee)
  else if count >= 2 && count <= 4 then do
    addLine "  sub sp, sp, #16" -- allocate 16 bytes for up to four args
    forM_ (zip args [0..4]) $ \(arg, i) -> do
      emit arg
      addLine ("  str r0, [sp, #" <> (toText (4 * i)) <> "]") -- store each arg in stack, offset in multiples of 4 
    addLine "  pop {r0, r1, r2, r3}" -- pop args from stack into registers
    addLine ("  bl " <> callee)
  else
    throwError (Unsupported "More than 4 arguments not supported")
```

We have an optimization for functions of zero or one arguments, in those cases there is no need to use the stack. Otherwise we do as described above, and throw and `Unsupported` error if there are more than four arguments.

### Function Definitions

Next we will actually be able to use these function calls by implementing function definitions. Defining a function has three main parts: the prologue, the function body, and the epilogue. For any function, we will have to perform the same prologue and epilogue to set up and tear down the function correctly.

Before executing the function body, we need to save the current values of two important registers: the frame pointer `fp` and the link register `lr`. We've already seen `fp`, which holds the address in the stack allocated for the current function call. The link register could also be called the return address, and tells us where to go back to after we call the function. 

So in our function prologue, we push `fp` and `lr` onto the stack to save their current values, then set the frame pointer to the current stack pointer. Then we save the current values in the argument registers `r0` through `r4`, since these are about to be clobbered with new values in the function body. The prologue looks like this:

```hs
addLine "  push {fp, lr}"  -- save the current frame pointer and link register 
addLine "  mov fp, sp"     -- set new frame pointer to current stack pointer
addLine "  push {r0, r1, r2, r3}" -- save argument registers
```

In the epilogue we essentially undo our work in the prologue. We deallocate the stack space we used in the current frame by moving `fp` into `sp`. Then, in case our function did not have a return statement, we implicitly return 0 by moving 0 into `r0`. Finally we restore the previous `fp` and pop the link register into the program counter register, `pc`, which sets us back to where we were in the program before the function. It looks like this:

```hs
addLine "  mov sp, fp"   -- deallocate stack space used for current frame
addLine "  mov r0, #0"   -- implicitly set return value to 0
addLine "  pop {fp, pc}" -- restore fp, pop the saved link register into pc to return 
```

In between these we need to emit the body. To execute the body, we need to set up a new local environment loaded with the parameters to the function. We do that in a function called `bindParams`, which takes a list of parameters and returns a new `Context` with the updated map of `locals` and `nextLocalOffset`. We store each paramter in slots of four bytes from the frame pointer:

```hs
bindParams :: [Text] -> Emit Context
bindParams params = do
  let offsets = [4 * i - 16 | i <- [0..]]
  setLocals (Map.fromList (zip params offsets))
  setNextLocalOffset (-20) -- 4 bytes after the allocated 4 params
  get
```

We actually return the `Context` wrapped in the `Emit` monad, since we want to keep the previous `labelCounter` preserved. That should continue to tick up between different function definitions, since otherwise we'll get repeat labels.

We need one more helper function, which we call `withContext`. This function takes two arguments, a context and some `Emit` action. All it does is save the value of the old context, put the new context, run the action, then restore the old context. This makes it so we can "pass" a fresh context to an action without clobbering the old one. Again, we avoid replacing the context wholesale to preserve that peksy `labelCounter` between function calls. In the end it looks like this:

```hs
withContext :: Context -> Emit () -> Emit ()
withContext ctx action = do
  Context{locals, nextLocalOffset} <- get
  put ctx
  action
  setLocals locals
  setNextLocalOffset nextLocalOffset
```

With all of the in place, all we need to do is put them together in our compilation of the function body, which is just this:

```hs
ctx <- bindParams params
withContext ctx (emit body)
```

Putting together the prologue, body, and epilogue, we finally arrive at the `emit` case for the AST node `Function`, which stores a function definition with its name, parameters and body:

```hs
Function name params body -> do
  if length params > 4 then 
    throwError (Unsupported "More than 4 params not supported")
  else do
    addLine ""
    addLine (".global " <> name)
    addLine (name <> ":")
    -- prologue
    addLine "  push {fp, lr}"  -- save the current frame pointer and link register
    addLine "  mov fp, sp"     -- set new frame pointer to current stack pointer
    addLine "  push {r0, r1, r2, r3}" -- save argument registers
    -- body
    ctx <- bindParams params
    withContext ctx (emit body)
    -- epilogue
    addLine "  mov sp, fp"   -- deallocate stack space used for current frame
    addLine "  mov r0, #0"   -- implicitly set return value to 0
    addLine "  pop {fp, pc}" -- restore fp, pop the saved link register into pc to return 
```

Again, we don't support more the 4 parameters, so we throw an error in that case. Otherwise we just create a new label for the function, and under that label put the code we've discussed above.

### Return

We implement a `Return` AST node exactly as we implemented the function definition prologue, so all we need is

```hs
Return term -> do
  emit term
  addLine "  mov sp, fp"   -- reset the stack pointer to current frame pointer 
  addLine "  pop {fp, pc}" -- reset fp, pop lr into pc to return
```

### Blocks

The last thing we'll look at is how to compile blocks. This is the simplest of all, since we just need to emit each statement contained in the block:

```hs
Block stmts ->
  forM_ stmts emit
```

## Testing

And with that, we've implemented a Turing-complete language. We can store values in memory, and we can loop easily enough with recursion. Putting everything together we can look at a side-by-side comparison of a simple factorial program and the assembly that the compiler generates:

```js
function main() {
  assert(factorial(5) == 120);
}

function factorial(n) {
  if (n == 1) {
    return 1;
  } else {
    return n * factorial(n - 1);
  }
}

function assert(x) {
  if (x) {
    putchar(46); // The character '.'
  } else {
    putchar(70); // The character 'F'
  }
}
```

When we assemble the generated assembly for this program, `gcc` will automatically link against `libc`, which gives us access to its standard functions like `putchar`. Here's the full assembly this produces:

```txt
.global main
main:
  push {fp, lr}
  mov fp, sp
  push {r0, r1, r2, r3}
  ldr r0, =5
  bl factorial
  push {r0, ip}
  ldr r0, =120
  pop {r1, ip}
  cmp r1, r0
  moveq r0, #1
  movne r0, #0
  bl assert
  mov sp, fp
  mov r0, #0
  pop {fp, pc}

.global factorial
factorial:
  push {fp, lr}
  mov fp, sp
  push {r0, r1, r2, r3}
  ldr r0, [fp, #-16]
  push {r0, ip}
  ldr r0, =1
  pop {r1, ip}
  cmp r1, r0
  moveq r0, #1
  movne r0, #0
  cmp r0, #0
  beq .L1
  ldr r0, =1
  mov sp, fp
  pop {fp, pc}
  b .L2
.L1:
  ldr r0, [fp, #-16]
  push {r0, ip}
  ldr r0, [fp, #-16]
  push {r0, ip}
  ldr r0, =1
  pop {r1, ip}
  sub r0, r1, r0
  bl factorial
  pop {r1, ip}
  mul r0, r1, r0
  mov sp, fp
  pop {fp, pc}
.L2:
  mov sp, fp
  mov r0, #0
  pop {fp, pc}

.global assert
assert:
  push {fp, lr}
  mov fp, sp
  push {r0, r1, r2, r3}
  ldr r0, [fp, #-16]
  cmp r0, #0
  beq .L3
  ldr r0, =46
  bl putchar
  b .L4
.L3:
  ldr r0, =70
  bl putchar
.L4:
  mov sp, fp
  mov r0, #0
  pop {fp, pc}
```

## Afterword

That's really all I have. If you want to try out the compiler yourself or dig around with the full source, [you can find it on GitHub here](https://github.com/micahcantor/comp-to-assembly-from-scratch-hs). I have a few more things implemented in the compiler than I showed here, like while loops, assignment statements, and array literals. 

If you happen to have a Raspberry Pi running a 32-bit OS, then you can run the generated assembly natively. Otherwise, you'll need to cross compile and emulate from your machine. On Debian/Ubuntu, download the packages `gcc-arm-linux-gnueabihf` and `qemu-user` then use the shell scripts included in the repository. 

I realized while writing this post that it would essentially be a worse version of Keleshev's book, since I wanted to give a tour of the compiler that made sense but also couldn't go into nearly as much detail as he did. So if you could follow along here, you'll probably find his book more enlightening. 

It's not a perfect book, but it gives a great introduction to understanding and compiling to assembly that is hard to find in a lot of other places. In the second part of the book, which I don't cover here, he looks at some more advanced concepts like garbage collection, type checking, and working with heap objects.

Anyway, if you found this interesting, follow me on Twitter at [@micah_cantor](https://twitter.com/micah_cantor) or feel free to reach out via [email](mailto:hello@micahcantor.com).

