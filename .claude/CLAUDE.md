# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`eliot.test` — a unit-testing framework for the [Eliot language](https://github.com/robertbraeutigam/eliot),
written *in* Eliot. It dogfoods: the library under `src/` is exercised by tests under `test/` that
are written with the very framework they test.

When reading or editing any `.els` file, use the `eliot-code` skill — it is the full language
reference. Eliot is total-by-default (no recursion/loops in user code), effects are written in
direct style, types are values, and there is one `Int` whose range is compiler-tracked
meta-information.

> **Effects v6 (2026-09-09).** There is **no carrier** in Eliot any more, and none in this framework.
> An effect is an ability declared with the `effect` keyword; an **implementation is a name** bound by
> `with` and forwarded lexically from `main` inward; a computation in a slot is a thunk whose operations
> were bound where it was written. Nothing here declares a carrier generic, and the words `Id`, `Suspend`,
> `Effect[…]` and "fake carrier" no longer mean anything. The compiler's `docs/effects.md` Part I is the
> authority.

## Architecture

A file is a module: `src/eliot/test/Test.els` is module `eliot.test.Test`. The whole framework is
three tiny modules plus the mocking library and a self-test:

- `eliot.test.Test` — the DSL, and **plain data throughout**. A **test case** is `data TestCase(subject:
  String, shouldPhrase: String)`: what it is about and what it should do, with *no body*. What running a
  body answered is `type Outcome = Either[AssertionError, Unit]`, and the two together are `data
  TestResult(testCase: TestCase, outcome: Outcome)`. **Nothing here stores a computation**, so the
  framework holds verdicts, not suspended work.

  **`type Test = {Writer[List[TestResult]]} Unit` is what a suite is**, and naming it is the whole of what a
  suite declares:

  ```eliot
  def testCases: Test = { … }                    -- cases that assert and nothing else
  def testCases: {Console} Test = { … }          -- cases that may print, for real
  ```

  `Test` is a **row alias** — a type alias whose body is a row. It lowers to `type Test = Unit` (a row is
  declaration metadata, never a type), and what crosses a use of it is the row's *entries*: a definition returning
  `Test` receives exactly what `{Writer[List[TestResult]]} Unit` would — the same declared row, the same "performs
  but does not declare" check. Being an ordinary name it crosses files, honours import scope and is shadowed by a
  local declaration; being a *return*-position alias it **composes** with a written-out row, which is how a suite
  widens what its cases may do. It works in return position **only**, which is why `in`'s and `runSuite`'s body
  slots still write `{Writer[List[TestResult]]} Unit` out longhand — a parameter row is *supplied* rather than
  received.

  A suite is a `Writer` computation that *accumulates* a `List[TestResult]` log. It is a `Writer`, not `State`,
  because a suite only ever *appends* cases and never reads back what has been registered — the honest, minimal
  contract for a collector (`Writer` = append-only `State`; the log's monoid is `Combine[List]` = concatenation).

  Two infix combinators build a suite in direct style:
  - `"what" should "acceptance"` → a `TestCase` (`infix none def should`). Naming a case is its own record, so
    `TestResult` nests it rather than repeating its fields.
  - `definition in body` → runs the body and registers its verdict (`infix none below should def in`). A `{ … }`
    block of `in` statements *sequences*: each `tell` contributes a singleton `[TestResult]`, the blocks' logs join
    via `Combine[List]`, so the suite collects **every** case in declaration order.

  **`in` supplies the assertion effect and nothing else**, and it is one line:

  ```eliot
  def in(testCase: TestCase, body: {Throw[AssertionError]} Unit): Test =
     tell(singleton(TestResult(testCase, runThrow(body))))
  ```

  `{Throw[AssertionError]}` on the slot is the one entry `in` *supplies*, so it is discharged per case — a
  failing assertion stops that case and no other. Every other operation the body performs is bound where it is
  written: by the suite's own declarations, or by a `with` the body itself carries. So what a test may do is
  decided by the suite's return type and by nothing in `in`. The body is a *slot*, not stored data, so no test is
  ever collected unrun.

  Three things then decide what a case may do, and they compose freely in one suite:
  - the **suite's return type** — the default; a bare `Test` admits assertions alone, `{Console} Test` admits
    `in { printLine(…); … shouldBe … }` performing for real, written inline with no definition of its own;
  - **`in mocked { … }`** — runs the case against `eliot.test.Mock`'s doubles for the base effects, with no
    fixture of any kind (see "Mocking");
  - **an author's own word** — a def of the author's own, binding their own named implementation on its body
    slot, which is what a project's *own* effect needs (see "Testing effectful code"). The framework's `mocked`
    is this shape; nothing in this repo needs a second one, so there is no worked example of it here.

- `eliot.test.Assertion` — `data AssertionError = Failed | NotEqual | UnexpectedlyEqual | NoErrorRaised`, a sum
  of *failure shapes* carrying the already-rendered values (assertions `show` at the raise site, where the
  instance is known; how a shape is presented is the runner's job). `Eq[AssertionError]` is structural — same
  shape, same values — so `expect` can check for a specific failure without coupling tests to presentation;
  `Show[AssertionError]` is a compact one-line debug rendering used when an error is itself a compared value.
  **Assertions signal failure by raising `AssertionError` through the `Throw` effect**; a passing assertion
  returns `unit`. This is why a body carries `{Throw[AssertionError]}`. Provides `success` (never raises),
  `fail(reason)`, the infix `actual shouldBe expected` and `actual shouldNotBe unexpected` (over any `Eq &
  Show`, raising `NotEqual` / `UnexpectedlyEqual`), `expect(error, body)` (passes iff `body` raises exactly
  `error`), and the infix `body describedAs newMessage` (`infix left below shouldBe` — rewrites a failing
  body's message).

  It also holds `shouldBeTrue`/`shouldBeFalse` (`Bool` has no `Show`, so `shouldBe true` does not compile)
  and `describedAs` is named that, not `message`, because `eliot.file.File` exports a `message` too and no
  file may import both.

  **`expect` and `describedAs` compose with any case.** `expect`'s body row *supplies* `Throw[E]`, saying
  nothing about anything else, so it wraps a body that performs a real effect, one running against doubles, or
  one that performs nothing. `describedAs` names the effect it raises itself (`{Throw[AssertionError]}` in and
  out) — which is fine and was not before v6: discharge is a **frame** installed at the `runThrow` call, and
  the nearest enclosing frame is its own, so the body's raise is caught here and the re-raise leaves through
  the enclosing case's frame.

- `eliot.test.Runner` — `def main: {Console} Unit`, the executable entry point.
  `foldNamedValues("testCases", noFailures, runSuite)` gathers every suite (see reflection below); `runSuite`
  discharges the Writer to its accumulated list (`runWriterToLog(suite)`) and **groups it by `subject`**
  (`List.groupBy`, so subjects appear in order of first mention). **One line is printed per subject, not per
  case**: green `✔ subject · N passed` while everything passed, red `✗ subject · N passed, M failed` as soon as
  anything did not, with each failure's `should` phrase and detail lines indented underneath.

  The run closes with **one `summary` line for everything that ran**: green
  ` ✔ all clear · N passed ` while every case passed, and an **inverted red band** ` ✗ M failed · N passed `
  (`ansiAlarm` — white on red) as soon as any did not, so the verdict is legible even after the per-subject
  report has scrolled. This is why the run functions **answer their cases' failure lines instead of `Unit`**:
  `runSuite`/`runSubject` return `List[List[String]]`, `main` flattens them, and `passedCount` (the shared
  "no lines = passed" count) serves the subject headers and the summary alike — one traversal, one convention,
  no second pass over the cases.

  `runSuite` receives each suite as a computation in a slot it **declares** —
  `suite: {Writer[List[TestResult]]} Unit` — and the suites after it as `rest: {} List[List[String]]`, still
  unrun. The slot supplies only `Writer`, the one effect `runSuite` discharges; every other effect a suite
  performs was bound where the suite was written. It writes the row out rather than naming `Test`, because a row
  alias works in return position only. So suites run in gathered order and each one's own output precedes its own
  report.

  The reporting is a pure-then-print split, and deliberately so: **the runner discharges nothing beyond the
  suite's `Writer`** — each body ran at its `in`, so `failureLines` merely folds `result.outcome` into that
  case's failure lines, empty exactly when it passed. `report` derives both the counts and the detail body from
  the one list, and `runSubject` is the only thing that prints. Keep the fold's body pure and
  `.foreach(printLine)` the lines it yields — which is why `runSubject` reads its group through the pure
  `groupSubject`/`groupResults` projections (the stdlib's `keyOf` trick) rather than printing inside a
  `foldPair`. `subjectOf` is the separate grouping key, reaching through `result.testCase.subject`.
  `describe` is the one place turning an `AssertionError` shape into presentation lines, and the `ansi*` helpers
  the one place holding an escape sequence. `header` picks its line with `fold`, **not** `if..else`: `else` and
  `++` have no declared relative precedence, so an `if..else` whose arms concatenate strings does not compile.

**The one architectural idea worth internalizing: tests register by name, via compile-time
reflection — there is no central list, no annotations, no import wiring.** The runner calls
`foldNamedValues("testCases", noFailures, runSuite)` (from `eliot.compiler.Reflect`), which reifies *every*
top-level value literally named `testCases` across all modules on the compiler path as a right fold —
`runSuite(name₁, suite₁, runSuite(name₂, suite₂, noFailures))`. **Reflection reifies code, not data**: each
gathered suite is an *argument* of a slot `runSuite` declares, never an element stored in a list, which is
exactly what lets a suite be a computation. `namedValues` remains, as the same fold at the free monoid, for
gathering plain data.

To add tests, declare `def testCases: Test = { "…" should "…" in { … } … }` in any module inside a compiled
source root, widening the row to what those tests need — `{Console} Test` to perform console effects for real.
`test/eliot/test/BasicAssertionsTests.els` is the worked example for plain cases, and
`test/eliot/test/example/GreeterTests.els` for the effectful ones. A suite is picked
up simply by being on the path; nothing references it. Suites are folded in qualified-name order, so a run is
reproducible. (One `testCases` per module — the reflection gathers one value per module under that name.)

> **Watch the build cache.** The compiler caches facts in the output directory (`target/.eliot-*`), and a
> `NamedValuesIndex` was once observed surviving a module's addition, so a newly added suite silently did not
> run while the build reported success — tests that do not run look exactly like tests that pass. It has not
> reproduced since; if a suite you just wrote does not appear in the report, `rm -rf target/.eliot-*` and rebuild
> before looking anywhere else.

- `eliot.test.Mock` — **the doubles, and the vocabulary a test writes.** One **named implementation per base
  effect** — `mockConsole`, `mockLog`, `mockFileSystem`, `mockProcess`, `mockEnvironment` — and one journal,
  `State[Recording]`, that their clauses write to. A test writes **no fixture**: it arranges, acts and asserts
  in one `in mocked { … }` block.

  `mocked` is the whole binding, written out on its slot's type:

  ```eliot
  def mocked(
        body: {Console, Log, FileSystem, Process, Environment, Mocking, Calls, Throw[IoError], Throw[AssertionError]} Unit
           with mockConsole with mockLog with mockFileSystem with mockProcess with mockEnvironment
     ): {Throw[AssertionError]} Unit =
     runStateToValue(emptyRecording, catch[IoError, Unit](body, e -> fail("unexpected I/O failure: " ++ show(e))))
  ```

  Read it as: the slot supplies those effects to the body, five of them bound to doubles by name, and the
  journal (`State[Recording]`) is discharged here so it never appears in a test's own row. An effect nothing
  mocks is not bound by this chain, so it is a compile error rather than a test that quietly does I/O.
  `Throw[IoError]` — which `FileSystem` and `Process` declare though no double ever raises one — is caught and
  reported as a failed case rather than left for a test to discharge by hand. The `catch`'s type arguments are
  spelled because the slot's row names `Throw` **twice**, and only the call can say which one is discharged.

  Arranging is an effect (`effect Mocking`), which is why `mocked` takes only a block: `whenSpawning`
  (with `succeeding`/`failing`/`exiting`), `whenSpawningCreates`, `whenReading`, `withFile`,
  `withDirectory`, `withVariable`, `withArguments`, `withWorkingDirectory`. **The most recent arrangement
  wins**, so a body may act, re-arm and act again, and a shared arrangement is an ordinary `{Mocking} Unit`
  definition — this framework's `@Before`, with no annotation behind it.

  Verifying is an effect too (`effect Calls`): `calls`, `callsMatching`, `lastCall`, `callCount`,
  `wasCalled`, `wasNeverCalled`, `wasCalledOnce`, `wasCalledTimes`, `wasCalledAtLeast`, `wasCalledAtMost`,
  `wereCalledInOrder`, `nothingWasCalled`, `onlyTheseWereCalled`, `forgetCalls`. Matching is by
  **containment** of the recorded line. `raising[E ~ Show](report, { … })` expects a failure — an ordinary
  top-level def now, not an operation of `Mocking`, so it works for a project's own error type as well as for
  `AssertionError`.

  **Why the framework owns this and a project cannot** — the v5 reason is gone (an implementation no longer
  needs a type to hang on, and a named one is never searched, so a project *could* now write its own doubles
  freely). What the framework still owns is the **work**: five doubles, one journal and one binding chain,
  written once. What a project writes is a double for *its own* effects.

## Testing effectful code

There are three ways to give a case a body, and all three register into the same suite.

### Real effects, inline — the default

A suite's return type says what its tests may do. Declare the effect and write the body inline; the
implementation is whatever the `Runner` binds at `main`:

```eliot
def testCases: {Console} Test = {
   "real effects" should "be available to a body whose suite declares them" in {
      printLine(" | (printed by a test performing a real Console effect)")
      success
   }
}
```

A failed assertion stops **that case** — `in` discharges `Throw[AssertionError]` per case — and the next case
still runs. A bare `Test` suite rejects a body that performs ("This value performs the effect 'Console' but does
not declare it").

> **A row cannot be closed.** There is no way to say "this body may perform *nothing*, whatever the suite
> allows". That is what `in pure { … }` meant, and it is **deleted**: a slot's row says what it *supplies*, not
> what it forbids, and an entry it does not supply continues the walk into the caller's scope. Making it
> expressible would be a language addition. A case that should perform nothing simply performs nothing.

### A named implementation, for determinism

Production code that declares `{Console}` can equally run against a double, because **the implementation is the
injection point**. The test declares its own named `implement`; production code is untouched and names nothing.
`test/eliot/test/example/` is the worked example of a project's side of that: `Greeter` is the application under
test and `GreeterTests` registers its pure, mocked and real-effect cases in **one** suite. The `Terminal` sketch
below is illustrative — for a *base* effect the doubles are already written (see "Mocking"), so nothing in this
repo needs to declare its own.

```eliot
effect Terminal {
   def write(line: String): Unit
   def read: String
}

def greet: {Terminal} Unit = {                  // production code
   val name = read
   write("Hello, " ++ name ++ "!")
}

implement session: Terminal {                   // the test's double, in the test module
   def write(line: String): {Writer[String]} Unit = tell(line ++ ";")
   def read: String = "Bob"
}

def greetTranscript: String = runWriterToLog(greet with session)
```

A double **cannot cheat**: a user module declares no natives, so an implementation reaches the world only
through effects **its own clauses declare** — which are charged, and bound, at the binding site. A double
**keeps its own state through an effect** (`Writer[String]` above), discharged where it is bound, never
appearing in `greet`'s row. And interpretation is **per effect**: `body with mockConsole with mockFileSystem`
leaves everything else at its default.

### An author's own word

For a project's **own** effect there is no ready double, so the project writes the binding word itself. Give a def
a single `{…}`-rowed parameter with the `with` on its slot type and it reads as a bare word with a block — the
shape `mocked` has, and `mocked` is this repo's only instance of it:

```eliot
def onTerminal(body: {Terminal, Throw[AssertionError]} Unit with session): {Throw[AssertionError]} Unit = …

"greet" should "greet whoever the terminal offers" in onTerminal {
   greet
   transcript shouldBe "Hello, Bob!;"
}
```

Assertions and faked effects interleave freely — there is no stacking, no lifting and no region rule, all of
which were carrier artefacts.

## What does not work

- **A row cannot be closed** (above) — `pure { … }` has no v6 spelling.
- **Sometimes a discharger's type arguments must be spelled.** The compiler reads a supplied entry's arguments
  off the *actual's declaration*; where nothing answers it refuses to guess ("Cannot tell which 'Throw' this
  call supplies… Write it out at the call"). This framework hits all three shapes: the actual is a **parameter**
  (`runThrow[AssertionError, Unit](body)` in `describedAs`, `runThrow[E, Unit](body)` in `raising`), the actual
  **raises nothing** (`raising[AssertionError]("expected", printLine(…))` in `MockVerificationTests`), or the
  slot's row names the **same ability twice** (`catch[IoError, Unit](…)` in `mocked`).
- **A computation may not be `val`-bound or dot-chained before being discharged.** Both are rowless positions,
  so the call *runs* there and the enclosing def is charged with the effect. Pass it to the discharger directly.
- **`import eliot.collection.List` shadows `Effect`'s `map`/`flatMap`** — no longer a hazard here, since there
  is no `eliot.carrier` and nothing to shadow.
- **A stored computation's binding is fixed where it is constructed.** A `with` applied to it later is an
  error, not a rebinding.

The framework compiles and runs: `src` + `test` builds `target/Runner.jar`, which prints one `✔`/`✗` line
per test subject with its pass and failure counts, then a closing summary line for the whole run. **96 cases,
all passing.**

> **Compiler version.** Needs a compiler at or past `2c3db71` (2026-09-12) — effects v6, the fix for an
> under-applied ability-implementation native (`32406522`, which `MockFileSystemTests`' `listDirectory(…).map(show)`
> hits), and the **row alias reached by ordinary name resolution**, which is what lets `Test` be declared in
> `eliot.test.Test` and named from a suite in another file.

## Building and running (compiler CLI)

There is no build system in *this* repo. Compilation is driven by a sibling checkout of the Eliot
compiler (see `eliot.paths` for its location — `/home/robert/personal/eliot`), whose `examples.run`
Mill task auto-appends the `lang`/`stdlib`/`jvm` layer source roots. You pass this project's own
roots as positional arguments:

```bash
cd /home/robert/personal/eliot          # the compiler checkout
./mill examples.run jvm exe-jar -m eliot.test.Runner \
   /home/robert/personal/eliot-test/src \
   /home/robert/personal/eliot-test/test \
   -o /home/robert/personal/eliot-test/target
java -jar /home/robert/personal/eliot-test/target/Runner.jar   # runs the discovered tests
```

Argument ordering is strict (scopt): `-m <module>` must come **immediately after `exe-jar`**, before
the positional source roots (once positional roots are consumed the subcommand scope is lost and
`-m` errors as "Unknown option"). The output flag `-o <dir>` trails at the end. The module for `-m`
is **fully qualified** — `eliot.test.Runner`, not `Runner`.

Every source root that should contribute tests must be passed. Passing both `src` and `test` runs the
framework's own self-tests. A downstream project using this framework passes `src` (the framework)
plus its own test root instead.

### `eliot.paths` — LSP only

`eliot.paths` lists all source roots (this project's `src`/`test` plus the base/stdlib/jvm layer
roots, and the `compiler`-pool overlays). **Only the IntelliJ LSP reads it** — in the IDE, "Run main"
on `eliot.test.Runner` builds and runs with no arguments. The compiler CLI ignores `eliot.paths`
entirely and requires every root as an explicit path argument; the `examples.run` task above supplies
the layer roots, and you supply `src`/`test`. Keep `eliot.paths` in sync with the CLI invocation if
you change either.
