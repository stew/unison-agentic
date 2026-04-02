# Unison Agent Instructions

## What Makes Unison Different

Unison is a statically typed functional language where **code is not stored in files**.
The authoritative source for all Unison code is the **UCM codebase**, accessed through
the **Unison MCP server**. This is unlike every other language — internalise it before
doing anything.

Key differences from Haskell/Elm/other functional languages that LLMs confuse:
- **No source files** — `.u` files are temporary sketches, never source of truth
- **No typeclasses** — use explicit dictionary passing (records of functions)
- **Strict evaluation** — not lazy by default
- **No `where` clauses** — define helpers in a block before the expression that uses them
- **No pattern matching on the left of `=`** — use `cases` or `match ... with`
- **No `let ... in`** — `let` introduces a block; there is no `in`
- **No `Nothing`/`Just`** — it's `Optional.None` / `Optional.Some`

## Core Rules

1. **MCP is the source of truth** — use `mcp__unison__view-definitions` to read code,
   `mcp__unison__update-definitions` to write it.
2. **Read before writing** — explore the project with MCP before adding anything.
   Understand naming conventions and existing patterns first.
3. **Always work on a branch** — never commit to `/main` directly. Ask the user for a
   branch name before calling `mcp__unison__update-definitions` for the first time.
   Only skip this if the user explicitly says to commit to `/main`.
4. **Typecheck, then commit** — sketch in `mcp__unison__typecheck-code`, fix errors,
   then commit with `mcp__unison__update-definitions`.
5. **Run tests after changes** — use `mcp__unison__run-tests`.
6. **Never treat `.u` files as durable source code** — except for the review file you
   produce at task completion (see below). All other `.u` files are scratch space only.

## Workflow for Adding/Updating Code

```
0. Ask the user: "What should I name the branch for this work?"
   Then: mcp__unison__create-branch            — create it before touching any code

1. mcp__unison__get-current-project-context   — orient yourself (confirm you are on the branch)
2. mcp__unison__view-definitions              — read related code
3. mcp__unison__typecheck-code                — verify your sketch
4. mcp__unison__update-definitions            — commit typechecked code to the branch
5. mcp__unison__run-tests                     — confirm nothing broke
6. When done: write the review file (see Task Completion below)
```

Track which definition names you add or modify throughout the session — you will need
them to produce the review file at the end.

## Most Common Mistakes (MUST AVOID)

### 1. Pattern matching on the left of `=`
```
-- WRONG (Haskell style):
List.head [] = None
List.head (x +: _) = Some x

-- RIGHT:
List.head = cases
  [] -> None
  x +: _ -> Some x
```

### 2. Multi-argument lambdas written as curried lambdas
```
-- WRONG:
List.zipWith (x -> y -> x + y) xs ys

-- RIGHT:
List.zipWith (x y -> x + y) xs ys
```

### 3. `where` clauses
```
-- WRONG:
f x = go x
  where go n = n + 1

-- RIGHT:
f x =
  go n = n + 1
  go x
```

### 4. `let ... in`
```
-- WRONG:
f x = let y = x + 1 in y * 2

-- RIGHT:
f x = let
  y = x + 1
  y * 2
```

### 5. Record field access with dot notation
```
-- WRONG: employee.name
-- RIGHT: Employee.name employee
```

### 6. Building lists in reverse then reversing
```
-- WRONG: accumulate with +: then List.reverse
-- RIGHT: accumulate with :+ (append to end, O(1))
go acc = cases
  [] -> acc
  x +: xs -> go (acc :+ x) xs
```

### 7. Non-tail-recursive list functions
```
-- WRONG (stack overflow on large lists):
map f = cases
  [] -> []
  x +: xs -> f x +: map f xs

-- RIGHT (tail recursive with accumulator):
map f xs =
  go acc = cases
    [] -> acc
    x +: xs -> go (acc :+ f x) xs
  go [] xs
```

### 8. `Record { field = value }` construction syntax
```
-- WRONG: Point { x = 1, y = 2 }
-- RIGHT: Point.Point 1 2   (constructor function named after the type)
```

## Quick Syntax Reference

```
-- Type signature + definition
factorial : Nat -> Nat
factorial n = product (range 1 (n + 1))

-- cases (preferred when last arg is immediately matched)
f = cases
  [] -> ...
  x +: xs -> ...   -- prepend pattern (head)
  xs :+ x -> ...   -- append pattern (last)

-- match expression
f xs = match xs with
  [] -> ...
  x +: rest -> ...

-- Multi-arg lambda
(x y -> x + y)

-- Thunk: type '{IO} () ; body: do printLine "hello"

-- Tail recursion (only looping construct)
sum ns =
  go acc = cases
    [] -> acc
    x +: xs -> go (acc + x) xs
  go 0 ns

-- Ability polymorphism in higher-order functions
map : (a ->{g} b) -> [a] ->{g} [b]

-- List operators
xs :+ item    -- append to end, O(1)
item +: xs    -- prepend to start, O(1)
xs ++ ys      -- concatenate

-- Passing operators as arguments (prefix syntax)
List.foldLeft (Nat.+) 0 ns
```

## Guide Routing Table

Read the relevant guide **before** writing code. Do not guess at APIs.

| Task | Guide to read |
|------|---------------|
| Any Unison code at all | `guides/language.md` |
| Writing tests | `guides/testing.md` |
| Abilities and effect handlers | `guides/abilities.md` |
| Concurrent / distributed code | `guides/concurrency.md` |
| Documentation blocks (`{{ }}`) | `guides/documentation.md` |
| Authoritative language reference via MCP | `guides/context.md` |
| Unison Cloud deployment | `guides/cloud.md` |

## Session Workflow

1. Orient: `mcp__unison__get-current-project-context`
2. Explore: read adjacent definitions before writing anything new
3. Read the relevant guide(s) from the table above
4. **Ask the user for a branch name** before writing any code (see Core Rules)
5. Use **TodoWrite** to track multi-step work and the list of definitions you modify
6. After code changes: run tests, add doc blocks to public APIs
7. **Write the review file** when the task is complete (see Task Completion below)
8. Tell the user: branch name, what was done, what to check in the review file

## Task Completion

When you believe a task is done, produce a review file so a human can inspect the
changes before merging the branch back to `/main`.

**How to produce the review file:**

1. Collect every definition name you added or modified during the session.
2. For each, call `mcp__unison__view-definitions` to get its final source.
3. Write all definitions into a file named `<branchname>.u` in the working directory.

**Format of the review file:**

```
-- Branch: <branchname>
-- Project: <project/branch>
-- Summary: <one-line description of what changed>
--
-- Definitions changed:
--   <list of names>
--
-- Review these changes and run `merge <branchname>` in UCM to merge to /main.

<all definitions, one after another, with a blank line between each>
```

The file is for human reading only. It does not need to be a runnable script.
Do not delete it — the user will remove it after reviewing.
