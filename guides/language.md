# Unison Programming Language Guide

Unison is a statically typed functional language with a unique approach to handling effects and distributed computation. This guide focuses on the core language features, syntax, common patterns, and style conventions.

## Core Language Features

Unison is a statically typed functional programming language with typed effects (called "abilities" or algebraic effects). It uses strict evaluation (not lazy by default) and has proper tail calls for handling recursion.

## Function Syntax

Functions in Unison follow an Elm-style definition with type signatures on a separate line:

```
factorial : Nat -> Nat
factorial n = product (range 1 (n + 1))
```

## Binary operators

Binary operators are just functions written with infix syntax like `expr1 * expr2` or `expr1 Text.++ expr2`. They are just like any other, except that their unqualified name isn't alphanumeric operator, which tells the parser to parse them with infix syntax.

Any operator can also be written with prefix syntax. So `1 Nat.+ 1` can also be written as `(Nat.+) 1 1`. But the only time you should use this prefix syntax is when passing an operator as an argument to another function. For instance:

```
sum = List.foldLeft (Nat.+) 0
product = List.foldLeft (Nat.*) 1
dotProduct = Nat.sum (List.zipWith (*) [1,2,3] [4,5,6])
```

IMPORTANT: when passing an operator as an argument to a higher order function, surround it in parens, as above. Otherwise the parser will treat it as an infix expression.

### Currying and Multiple Arguments

Functions in Unison are automatically curried. A function type like `Nat -> Nat -> Nat` can be thought of as either:

- A function taking two natural numbers
- A function taking one natural number and returning a function of type `Nat -> Nat`

Arrow types (`->`) associate to the right. Partial application is supported by simply providing fewer arguments than the function expects:

```
add : Nat -> Nat -> Nat
add x y = x + y

add5 : Nat -> Nat
add5 = add 5  -- Returns a function that adds 5 to its argument
```

## Lambdas

Anonymous functions or lambdas are written like `x -> x + 1` or `x y -> x + y*2` with the arguments separated by spaces.

Here's an example:

CORRECT:

```
List.zipWith (x y -> x*10 + y) [1,2,3] [4,5,6]
```

INCORRECT:

```
List.zipWith (x -> y -> x*10 + y) [1,2,3] [4,5]
```

Once again, a multi-argument lambda just separates the arguments by spaces.

## Type Variables and Quantification

In Unison, lowercase variables in type signatures are implicitly universally quantified. You can also explicitly quantify variables using `forall`:

```
-- Implicit quantification
map : (a -> b) -> [a] -> [b]

-- Explicit quantification
map : forall a b . (a -> b) -> [a] -> [b]
```

Prefer implicit quantification, not explicit `forall`. Only use `forall` when defining higher-rank functions which take a universally quantified function as an argument.

## Algebraic Data Types

Unison uses algebraic data types similar to other functional languages:

```
type Optional a = None | Some a

type Either a b = Left a | Right b
```

### Unique types vs structural types

Unison has two kinds of user-defined types:

- **Unique types** (the default): `type Foo = ...` or equivalently `unique type Foo = ...`. Each definition gets a unique identity (a UUID baked in at definition time), so two types with the same structure are still distinct. This is almost always what you want.
- **Structural types**: `structural type Foo = ...`. These are compared purely by structure, like tuples. Two structural types with identical fields are interchangeable.

**Always use plain `type`. Never write `structural type`.**

Structural types make it easy to accidentally confuse two unrelated types that happen to have the same shape, and they offer no practical advantage in normal code. If you find yourself reaching for `structural type`, use a unique type instead.

Pattern matching works as you would expect:

```
Optional.map : (a -> b) -> Optional a -> Optional b
Optional.map f o = match o with
  None -> None
  Some a -> Some (f a)
```

#### Prefer using `cases` where possible

A function that immediately pattern matches on its last argument, like so:

```
Optional.map f o = match o with 
  None -> None
  Some a -> Some (f a)
```

Can instead be written as:

```
Optional.map f = cases 
  None -> None
  Some a -> Some (f a)
```

Prefer this style when applicable.

The `cases` syntax is also handy when the argument to a function is a tuple, to destructure the tuple, for instance:

```
List.zip xs yz |> List.map (cases (x,y) -> frobnicate x y "hello")
```

The `cases` syntax can also be used for functions that take multiple arguments. Just separate the arguments by a comma, as in:

```
-- Using multi-arg cases
merge : [a] -> [a] -> [a] -> [a]
merge acc = cases
  [], ys -> acc ++ ys
  xs, [] -> acc ++ xs
  -- uses an "as" pattern
  xs@(hd1 +: t1), ys@(hd2 +: t2)
    | Universal.lteq hd1 hd2 -> merge (acc :+ hd1) t1 ys
    | otherwise -> merge (acc :+ hd2) xs t2
```

This is equivalent to the following definition which tuples up the two arguments and matches on that:

```
-- Using pattern matching on a tuple
merge acc xs ys = match (xs, ys)
  ([], ys) -> acc ++ ys
  (xs, []) -> acc ++ xs
  (hd1 +: t1, hd2 +: t2)
    | Universal.lteq hd1 hd2 -> merge (acc :+ hd1) t1 ys
    | otherwise -> merge (acc :+ hd2) xs t2
```

#### Rules on Unison pattern matching syntax

VERY IMPORTANT: the pattern matching syntax is different from Haskell or Elm. You CANNOT do pattern matching to the left of the `=` as you can in Haskell and Elm. ALWAYS introduce a pattern match using a `match <expr> with <cases>` form, or using the `cases` keyword.

INCORRECT (DO NOT DO THIS, IT IS INVALID):

```
List.head : [a] -> Optional a
List.head [] = None 
List.head (hd +: _tl) = Some hd
```

CORRECT: 

```
List.head : [a] -> Optional a
List.head = cases
  [] -> None
  hd +: _tl -> Some hd
```

ALSO CORRECT: 

```
List.head : [a] -> Optional a
List.head as = match as with
  [] -> None
  hd +: _tl -> Some hd
```

### Important naming convention

Note that UNLIKE Haskell or Elm, Unison's `Optional` type uses `None` and `Some` as constructors (not `Nothing` and `Just`). 

Becoming familiar with Unison's standard library naming conventions is important.

Use short variable names for helper functions:

* For instance: `rem` instead of `remainder`, and `acc` instead of `accumulator`.
* If you need to write a helper function loop using recursion, call the recursive function `go` or `loop`.
* Use `f` or `g` as the name for a generic function passed to a higher-order function like `List.map`

## Lists

Here is the syntax for a list, and a few example of pattern matching on a list:

```
[1,2,3]

-- append to the end of a list
[1,2,3] :+ 4 === [1,2,3,4] 

-- prepend to the beginning of a list
0 +: [1,2,3] === [0,1,2,3]

[1,2,3] ++ [4,5,6] === [1,2,3,4,5,6]

match xs with
  [1,2,3] ++ rem -> transmogrify rem
  init ++ [1,2,3] -> frobnicate init
  init :+ last -> wrangle last
  [] -> 0
```

List are represented as finger trees, so adding elements to the start or end is very fast, and random access using `List.at` is also fast.

IMPORTANT: DO NOT build up lists in reverse order, then call `List.reverse` at the end. Instead just build up lists in order, using `:+` to add to the end.

## Use accumulating parameters and tail recursive functions for looping

Tail recursion is the sole looping construct in Unison. Just write recursive functions, but write them tail recursive with an accumulating parameter. For instance, here is a function for summing a list:

```
Nat.sum : [Nat] -> Nat
Nat.sum ns =
  go acc = cases
    [] -> acc
    x +: xs -> go (acc + x) xs
  go 0 ns
```

IMPORTANT: If you're writing a function on a list, use tail recursion and an accumulating parameter.

CORRECT:

```
List.map : (a ->{g} b) -> [a] ->{g} [b]
List.map f as = 
  go acc = cases
    [] -> acc
    x +: xs -> go (acc :+ f x) xs
  go [] as
```

INCORRECT (not tail recursive):

```
-- DON'T DO THIS 
List.map : (a ->{g} b) -> [a] ->{g} [b]
List.map f = cases
  [] -> []
  x +: xs -> f x +: go xs
```

## Built up lists in order, do not build them up in reverse order and reverse them at the end

Unison lists support O(1) append at the end of the list (they are based on finger trees), so you can just build up the list in order. Do not build them up in reverse order and then reverse at the end.

INCORRECT:

```
List.map : (a ->{g} b) -> [a] ->{g} [b]
List.map f as = 
  go acc = cases
    [] -> List.reverse acc
    x +: xs -> go (f x +: acc) xs
  go [] as
```

CORRECT:

```
List.map : (a ->{g} b) -> [a] ->{g} [b]
List.map f as = 
  go acc = cases
    [] -> acc
    x +: xs -> go (acc :+ f x) xs
  go [] as
```

Note that `:+` appends a value onto the end of a list in constant time, O(1):

```
[1,2,3] :+ 4
=== [1,2,3,4]
```

While `+:` prepends a value onto the beginning of a list, in constant time:

```
0 +: [1,2,3]
=== [0,1,2,3]
```

## Abilities (Algebraic Effects)

> **Note:** For comprehensive coverage of abilities, handlers, and effect management, see `unison-abilities-guide.md`.

Unison has a typed effect system called "abilities". The `{g}` notation in type signatures represents effects:

```
-- A function with effects
Optional.map : (a ->{g} b) -> Optional a ->{g} Optional b
```

Built-in abilities include `IO` for side effects and `Exception` for error handling. For concurrent code, see `unison-concurrency-guide.md` for the `Remote` ability.

## Record Types

Record types in Unison are defined as:

```
type Employee = { name : Text, age : Nat }
```

This generates accessor functions and updaters:

- `Employee.name : Employee -> Text`
- `Employee.age : Employee -> Nat`
- `Employee.name.set : Text -> Employee -> Employee`
- `Employee.age.modify : (Nat -> Nat) -> Employee -> Employee`

Example usage:

```
doubleAge : Employee -> Employee
doubleAge e = Employee.age.modify (n -> n * 2) e
```

### Important: Record Access Syntax

A common mistake is to try using dot notation for accessing record fields. In Unison, record field access is done through the generated accessor functions:

```
-- INCORRECT: ring.zero
-- CORRECT:
Ring.zero ring
```

Record types in Unison generate functions, not special field syntax.

## Namespaces and Imports

Unison uses a flat namespace with dot notation to organize code. You can import definitions using `use`:

```
use List map filter

-- Now you can use map and filter without qualifying them
evens = filter Nat.isEven [1,2,3,4]
incremented = map Nat.increment (range 0 100)
```

A wildcard import is also available:

```
use List  -- Imports all List.* definitions
```

## Collection Operations

List patterns allow for powerful matching:

```
-- Match first element of list
a +: as

-- Match last element of list
as :+ a

-- Match first two elements plus remainder
[x,y] ++ rem
```

Example implementation of `List.map`:

```
List.map : (a ->{g} b) -> [a] ->{g} [b]
List.map f as =
  go acc rem = match rem with
    [] -> acc
    a +: as -> go (acc :+ f a) as
  go [] as
```

### List Functions and Ability Polymorphism

Remember to make list functions ability-polymorphic if they take function arguments. This allows the function passed to operate with effects:

```
-- CORRECT: Ability polymorphic
List.map : (a ->{g} b) -> [a] ->{g} [b]

-- INCORRECT: Not ability polymorphic
List.map : (a -> b) -> [a] -> [b]
```

## Pattern Matching with Guards

Guards allow for conditional pattern matching:

```
List.filter : (a -> Boolean) -> [a] -> [a]
List.filter p as =
  go acc rem = match rem with
    [] -> acc
    a +: as | p a -> go (acc :+ a) as
            | otherwise -> go acc as
  go [] as
```

### Guard Style Convention

When using multiple guards with the same pattern, align subsequent guards vertically under the first one, not repeating the full pattern:

```
-- CORRECT:
a +: as | p a -> go (acc :+ a) as
        | otherwise -> go acc as

-- INCORRECT:
a +: as | p a -> go (acc :+ a) as
a +: as | otherwise -> go acc as
```

## Block Structure and Binding

In Unison, the arrow `->` introduces a block, which can contain multiple bindings followed by an expression:

```
-- A block with a helper function
dotProduct ring xs ys =
  go acc xsRem ysRem = match (xsRem, ysRem) with
    ([], _) -> acc
    (_, []) -> acc
    (x +: xs, y +: ys) ->
      nextAcc = Ring.add ring acc (Ring.mul ring x y)
      go nextAcc xs ys
  go (Ring.zero ring) xs ys
```

### Bindings in blocks

Inside a block (a function body, a `do` block, a `cases` arm body, etc.), write bindings as plain `name = expr`:

```
nextAcc = Ring.add ring acc (Ring.mul ring x y)
```

### `let` introduces a block (no `in`)

`let` exists in Unison and introduces a new block, but there is **no `in`** — the block continues via indentation, like a `do` block but without laziness:

```
-- CORRECT: let introduces a block, no `in`
f x = let
  y = x + 1
  y * 2

-- INCORRECT: let...in is not valid Unison
f x = let y = x + 1 in y * 2
```

### Multi-statement lambda bodies with `let`

A lambda `x -> expr` can only contain a single expression. To run multiple statements inside a parenthesised lambda, use `let` to introduce a block:

```
-- CORRECT: let inside a lambda gives a multi-statement body
List.foreach (x -> let
  line = "item: " ++ Text.show x
  printLine line) xs

-- ALSO CORRECT for a single trailing lambda at the end of a block
-- (trailing lambda syntax — only valid as the last statement):
List.foreach xs x ->
  doSomething x
  doSomethingElse x
```

**Key rules:**
- `(x -> let ...)` — multi-statement lambda anywhere; `let` introduces the block
- `f xs x -> body` trailing syntax — only valid as the **last** statement in a block; the body is a full block
- `=` bindings are **not** valid inside `(x -> ...)` without a `let`; extract a named helper or use `let`
- `do` makes a thunk `'{g} a`; `let` is an eager block — do not use `do` where you need a function `a ->{g} b`

### `if-then` without `else` in multi-statement blocks

In a multi-statement block, always write `else ()` explicitly when the `then` branch returns `()`. Without it, the parser may consume the next statement as the implicit `else` branch, causing confusing type errors or silent mis-parses.

```
-- DANGEROUS: the parser may treat `nextStatement` as the else branch
if cond then doSomething()
nextStatement

-- CORRECT: explicit else ()
if cond then doSomething()
else ()
nextStatement
```

This is especially important when there are further statements (or the function's final return expression) after the `if`.

### Zero-argument recursive thunks

When you need a recursive helper with no parameters that uses abilities (IO, Exception, etc.), define it as a `do` thunk and call it with `()`:

```
-- WRONG: `go` is a value binding, not a thunk; ability access fails
go =
  buf = Ref.read bufRef   -- needs {IO}
  go

-- CORRECT: `do` makes `go` a thunk of type '{IO} a
go : '{IO} Bytes
go = do
  buf = Ref.read bufRef
  go()    -- recursive thunk call
go()
```

The type annotation is optional but helps readability.

### Constructor pattern bindings require `match`, not `let`

You cannot use a top-level let-binding to destructure a single-constructor type with a fully-qualified constructor name. The pattern binding form `Constructor var = expr` does not bring `var` into scope reliably — the compiler treats it as an unresolved name.

**WRONG:**
```
http2.IncomingQueue.IncomingQueue qRef = q
-- qRef is NOT in scope below this line
```

**CORRECT:** use an explicit `match`:
```
match q with
  http2.IncomingQueue.IncomingQueue r -> go r
```

Or for single-use unwrapping, inline it:
```
go r = ...
match q with MyType.Constructor r -> go r
```

**Why:** Unison's let-pattern binding does not reliably introduce variables when the constructor name is qualified (dotted). Use `match` instead.

### Common type conversions

- `Nat.toInt : Nat -> Int` — convert a natural number to an integer (there is no `Int.fromNat`)
- `Text.fromUtf8 : Bytes ->{Exception} Text` — raises on invalid UTF-8
- `toUtf8 : Text -> Bytes`

### No `where` Clauses

Unison doesn't have `where` clauses. Helper functions must be defined in the main block before they're used.

```
-- CORRECT: declare the helper function, then use it later in the block
filterMap : (a ->{g} Optional b) -> [a] ->{g} [b]
filterMap f as = 
  go acc = cases
    [] -> acc
    (hd +: tl) -> match f hd with
      None -> go acc tl
      Some b -> go (acc :+ b) tl
  go [] as

-- INCORRECT
filterMap : (a ->{g} Optional b) -> [a] ->{g} [b]
filterMap f as = go [] as 
  where
  go acc = cases
    [] -> acc
    (hd +: tl) -> match f hd with
      None -> go acc tl
      Some b -> go (acc :+ b) tl
  go [] as
```

## Error Handling

Unison uses the `Exception` ability for error handling:

```
type Failure = Failure Link.Type Text Any

-- Raising an exception
Exception.raise (Failure (typeLink Generic) "An error occurred" (Any 42))

-- Catching specific exceptions
Exception.catchOnly : Link.Type -> '{g, Exception} a ->{g, Exception} Either Failure a
```

## Text Handling

Unison calls strings `Text` and uses concatenation with `++`:

```
greeting = "Hello, " Text.++ name
```

You can use `use Text ++` to use `++` without qualification:

```
use Text ++
greeting = "Hello, " ++ name
```

### No String Interpolation

Unlike many modern languages, Unison doesn't have string interpolation. Text concatenation with `++` is the primary way to combine text values.

## The Pipeline Operator `|>`

Unison provides the `|>` operator for creating pipelines of transformations. The expression `x |> f` is equivalent to `f x`. This is particularly useful for composing multiple operations in a readable left-to-right flow:

```
use Nat *
use List filter sum map

-- Calculate the sum of odd numbers after multiplying each by 10
processNumbers : [Nat] -> Nat
processNumbers numbers =
  numbers
    |> map (x -> x * 10)   -- Multiply each number by 10
    |> filter Nat.isOdd    -- Keep only odd numbers
    |> sum                 -- Sum all remaining numbers

-- Using the function with numbers 1 through 100
result =
  range 1 101            -- Create a list from 1 to 100 (inclusive)
    |> processNumbers      -- Apply our processing function
```

This style makes the code more readable by placing the data first and showing each transformation step in sequence, similar to the pipe operator in languages like Elm, F#, or Elixir.

## Writing documentation

Documentation blocks appear just before a function or type definition. They look like so:

````
{{
``List.map f xs`` applies the function `f` to each element of `xs`.

# Examples

```
List.map Nat.increment [1,2,3]
```

```
List.map (x -> x * 100) (range 0 10)
```
}}
List.map f xs = 
  go acc = cases
    [] -> acc
    hd +: tl -> go (acc :+ f hd) tl
  go [] xs
````

## Type System Without Typeclasses

Unison doesn't have typeclasses. Instead, it uses explicit dictionary passing:

```
type Ring a =
  { zero : a
  , one : a
  , add : a -> a -> a
  , mul : a -> a -> a
  , neg : a -> a
  }

dotProduct : Ring a -> [a] -> [a] -> a
dotProduct ring xs ys =
  go acc xsRem ysRem = match (xsRem, ysRem) with
    ([], _) -> acc
    (_, []) -> acc
    (x +: xs, y +: ys) ->
      nextAcc = Ring.add ring acc (Ring.mul ring x y)
      go nextAcc xs ys
  go (Ring.zero ring) xs ys
```

## Program Entry Points

Main functions in Unison can have any name:

```
main : '{IO, Exception} ()
main = do printLine "hello, world!"
```

The syntax `'{IO, Exception} ()` is sugar for `() ->{IO, Exception} ()`, representing a thunk that can use the `IO` and `Exception` abilities.

UCM (Unison Codebase Manager) is used to run or compile programs:

```
# Run directly
run main

# Compile to bytecode
compile main out

# Run compiled bytecode
ucm run.compiled out.uc
```

## Testing

Unison has built-in testing support:

```
test> Nat.tests.props = test.verify do
  Each.repeat 100
  n = Random.natIn 0 1000
  m = Random.natIn 0 1000
  labeled "addition is commutative" do 
    ensureEqual (n + m) (m + n)
  labeled "zero is an identity for addition" do
    ensureEqual (n + 0) (0 + n)
  labeled "multiplication is commutative" do
    ensureEqual (n * m) (m * n)
```

REQUIREMENT: Tests for a function or type `foo` should always be named `foo.tests.<test-name>`.

## Lazy Evaluation

Unison is strict by default. Laziness can be achieved using `do` (short for "delayed operation", NOT the same as Haskell's `do` keyword): 

```
-- Inside a function, use do
nats : '{Stream Nat} ()
nats = do 
  Stream.emit 1
  Stream.emit 2
  Stream.emit 3
```

You can use `(do someExpr)` to put a delayed computation anywhere you want (say, as the argument to a function), or if a `do` is the last argument to a function, you can leave off the parentheses:

PREFERRED: 

```
forkAt node2 do
  a = 1 + 1
  b = 2 + 2
  a + b
```

## Standard Library

Unison's standard library includes common data structures:

- `List` for sequences
- `Map` for key-value mappings
- `Set` for unique collections
- `Pattern` for regex matching

## Additional Resources

- Unison docs on regex patterns: https://share.unison-lang.org/@unison/base/code/main/latest/types/Pattern
- Testing documentation: https://share.unison-lang.org/@unison/base/code/main/latest/terms/test/verify
