# Unison Testing Companion for Test-Writing Agents

This guide is a practical companion to the Unison docs, focused on how an agent should write, run, and debug tests efficiently.

## 1. Mental model: what is special about Unison tests

- Pure unit tests are regular Unison terms with type `[test.Result]`.
- A `test>` watch expression is evaluated on file save and can be added to the codebase as a named term.
- `test` results are hash-cached by dependency graph. If dependencies have not changed, `test` may report cached results.
- Pure tests should not depend on nondeterministic abilities like `IO`.
- IO tests are handled separately via `io.test` / `io.test.all` (or by running test thunks directly with MCP run APIs).

## 2. Core test shapes

### 2.1 Minimal pure test with `test>` and `check`

```unison
square : Nat -> Nat
square x = x * x

test> square.tests.ex1 = check (square 4 == 16)
```

Notes:
- `check : Boolean -> [test.Result]` is the simplest assertion bridge.
- Follow the namespace convention `<subject>.tests.*`.

### 2.2 Property-style pure test with `test.verify`

```unison
textRoundtrip.tests.parsePrint : [test.Result]
textRoundtrip.tests.parsePrint = test.verify do
  Each.repeat 100
  n = Random.nat()
  test.ensureEqual (Nat.fromText (Nat.toText n)) (Optional.Some n)
```

`test.verify` runs a block that may use abilities such as `Random`, `Each`, `Exception`, and `Label`, then returns `[test.Result]`.

## 3. Assertion toolbox (what to reach for)

Common `test` assertions in `@unison/base`:

- `test.ensure` - assert boolean true.
- `test.ensureWith` - assert boolean true with custom payload.
- `test.ensureEqual` - assert equality.
- `test.ensureEqualWith` - assert equality using a custom comparator.
- `test.ensureNotEqual` - assert inequality.
- `test.ensureLess` / `test.ensureLessOrEqual`.
- `test.ensureGreater` / `test.ensureGreaterOrEqual`.
- `test.ensuring` - assert an effectful boolean computation returns true.
- `test.ensuringWith` - effectful boolean assertion with message/payload.
- `test.raiseFailure` - force an explicit failure with message + payload.

Important clarification:
- There is no standard `test.ensureFailure` in `@unison/base` (at least on current main).
- Use `test.raiseFailure` for explicit failures, or assert failure behavior by catching/handling exceptions in a test harness.

Example using several assertions:

```unison
math.tests.ordering : [test.Result]
math.tests.ordering = test.verify do
  x = 10
  y = 20
  test.ensureLess x y
  test.ensureGreater y x
  test.ensureNotEqual x y
  test.ensureEqual (x + y) 30
```

## 4. Generating strong test coverage

### 4.1 Randomized checks

Use `Each.repeat` and `Random` together:

```unison
reverse.tests.concatLaw : [test.Result]
reverse.tests.concatLaw = test.verify do
  use Random natIn
  use Text ++ reverse

  Each.repeat 100
  size1 = natIn 0 50
  size2 = natIn 0 50
  t1 = Text.ofChars Text.unicode size1
  t2 = Text.ofChars Text.unicode size2

  test.ensureEqual (reverse t1 ++ reverse t2) (reverse (t2 ++ t1))
```

### 4.2 Edge-first generation helpers

Prefer `arbitrary.*` helpers when available:
- `arbitrary.ints`
- `arbitrary.nats`
- `arbitrary.floats`
- `unspecialFloats`

These include corner cases first, then random values.

### 4.3 Deterministic coverage with `Each`

Use ranges/lists for small finite domains:

```unison
hex.tests.upperDigits : [test.Result]
hex.tests.upperDigits = test.verify do
  (n, c) = each (List.zip (Nat.range 10 16) (toCharList "ABCDEF"))
  test.ensureEqual (Nat.toTextBase 16 n) (Optional.Some (Char.toText c))
```

## 5. Failure diagnosis that scales

Use labels to keep failures actionable:

```unison
text.tests.emptyProps : [test.Result]
text.tests.emptyProps = test.verify do
  labeled "empty text" do
    labeled "head" do test.ensureEqual (Text.head "") Optional.None
    labeled "isEmpty" do test.ensure (Text.isEmpty "")
    labeled "size" do test.ensureEqual (Text.size "") 0
```

Use `label` to attach case payloads:

```unison
mul.tests.badAssumption : [test.Result]
mul.tests.badAssumption = test.verify do
  x = 0
  y = 0
  label "x * y should be greater than x" (x, y)
  test.ensureGreater (x * y) x
```

## 6. IO tests: patterns and execution

Pure `test` is not for `IO`. IO tests should be delayed computations returning `[test.Result]`.

### 6.1 `io.test>` watch expression example

```unison
io.test> file.tests.readNonEmpty = do
  contents = "hello"
  test.ensure (Text.size contents > 0)
  check true
```

### 6.2 Named IO test term + UCM command

```unison
file.tests.ioRead : '{IO, Exception} [test.Result]
file.tests.ioRead = do
  printLine "running IO setup"
  check (1 == 1)
```

Run in UCM:
- `io.test file.tests.ioRead`
- `io.test.all` (run all IO tests on current branch)

## 7. Running tests from MCP (agent automation)

### 7.1 Pure tests

Use MCP pure test runner:
- `mcp__unison__run-tests`
- Recommended for CI-like agent loops over a project or subnamespace.

Typical call shape:
- `projectContext = { projectName, branchName }`
- optional `subnamespace` to scope runs

### 7.2 IO tests

Use `mcp__unison__run` for specific IO test thunks when you need programmatic results.

Example strategy:
1. Keep IO tests as named terms with type `'{IO, Exception} [test.Result]`.
2. Invoke via `mcp__unison__run` using that term as `mainFunctionName`.
3. Parse output and fail agent step when any `Result.Fail` is present.

If you want branch-wide IO test execution semantics, mirror UCM `io.test.all` behavior in your agent orchestration.

## 8. Suggested agent workflow

1. Write smallest test that can fail for intended behavior.
2. Prefer pure `test.verify` first; move to IO tests only when behavior requires effects.
3. Include at least one edge-case set (`Optional.None`, empty text/list, zero, boundaries).
4. Add labels and payloads so failures are self-diagnosing.
5. Keep tests deterministic unless randomness is explicitly required.
6. When using randomness, combine corner-case generators + repeated randomized checks.

## 9. Common mistakes to avoid

- Mixing `IO` into pure `test>` tests.
- Forgetting that `test` may use cached results and assuming fresh execution happened.
- Writing assertions without labels/payloads, producing low-signal failures.
- Relying on only random data and missing deterministic edge cases.
- Assuming `ensureFailure` exists in base; use `raiseFailure` or explicit exception assertions instead.

## 10. Quick command cheat sheet

UCM:
- `test`
- `test <namespace>`
- `io.test <name>`
- `io.test.all`

MCP:
- `mcp__unison__run-tests` for pure test suites
- `mcp__unison__run` for explicit IO test terms

