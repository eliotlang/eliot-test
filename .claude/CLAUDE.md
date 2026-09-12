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

**There are no carriers here.** A computation is a thunk, an effect row is what a definition performs, and an
implementation is a *name* bound by `with` or by whoever runs the program. If you find a comment mentioning a
carrier, `Id`, `Suspend`, an `Effect` instance, a pinned row (`{… | G}`), a capture tag (`{| F}`) or `pure { … }`,
it is history — see "What the carrier model took with it" below.

## Architecture

A file is a module: `src/eliot/test/Test.els` is module `eliot.test.Test`. The whole framework is
three tiny modules plus a self-test:

- `eliot.test.Test` — the DSL, and **plain data throughout**. A **test case** is `data TestCase(subject:
  String, shouldPhrase: String)`: what it is about and what it should do, with *no body*. What running a
  body answered is `type Outcome = Either[AssertionError, Unit]`, and the two together are `data
  TestResult(testCase: TestCase, outcome: Outcome)`. Nothing here stores a computation — the framework holds
  verdicts, not suspended work.

  **`type Test = {Writer[List[TestResult]]} Unit` is what a suite is**, and it is the only thing a suite
  declares:

  ```eliot
  def testCases: Test = { … }                    -- cases that assert and nothing else
  def testCases: {Console} Test = { … }          -- cases that may print, for real
  ```

  `Test` is a **row alias** — a type alias whose body is a row. It lowers to `type Test = Unit` (a row is
  declaration metadata and never a type), and what crosses a use of it is the row's *entries*: a definition
  returning `Test` receives exactly what `{Writer[List[TestResult]]} Unit` would — the same binders, the same
  declared row, the same "performs but does not declare" check. Being an ordinary name it crosses files, honours
  import scope and is shadowed by a local declaration; being a *return*-position alias it **composes** with a
  written-out row, which is how a suite widens what its cases may do. It works in return position **only** — a
  row alias at a parameter or a field is a compile error, not a silent widening — which is why `in`'s and
  `runSuite`'s body slots still write `{Writer[List[TestResult]]} Unit` out longhand.

  A suite is a `Writer` computation that *accumulates* a `List[TestResult]` log. It is a `Writer`, not `State`,
  because a suite only ever *appends* cases and never reads back what has been registered — the honest, minimal
  contract for a collector (`Writer` = append-only `State`; the log's monoid is `Combine[List]` = concatenation).

  Two infix combinators build a suite in direct style:
  - `"what" should "acceptance"` → a `TestCase` (`infix none def should`). Naming a case is its own record, so
    `TestResult` nests it rather than repeating its fields.
  - `definition in body` → runs the body and registers its verdict (`infix none below should def in`). A `{ … }`
    block of `in` statements *sequences*: each `tell` contributes a singleton `[TestResult]`, the blocks' logs join
    via `Combine[List]`, so the suite collects **every** case in declaration order.

  **`in` supplies the assertion effect and nothing else.** Its body slot is `{Throw[AssertionError]} Unit` — an
  entry `in`'s own row lacks, so it is *supplied*, i.e. discharged right there, per case: a failing assertion
  stops that case and no other. Every other effect the body performs is charged outward, so it is the **suite's
  return type** that decides what a case may do, and nothing in `in` does. The body is a *slot*, not stored data,
  so it runs where it stands: no case is ever collected unrun.

  Two things then decide what a case may do, and they compose freely in one suite:
  - the **suite's return type** — `Test` admits assertions alone, `{Console} Test` admits printing, written
    inline with no definition of its own;
  - **an author's own word** — `in onConsole { … }` runs the body against test doubles the word binds on its
    slot's type (see "Testing effectful code").

  There is no third word forcing a case to be pure: a bare `Test` suite already is, because a row says what it
  supplies and a suite that supplies only registration admits only assertions.

- `eliot.test.Assertion` — `data AssertionError = Failed | NotEqual | UnexpectedlyEqual | NoErrorRaised`, a sum
  of *failure shapes* carrying the already-rendered values (assertions `show` at the raise site, where the
  implementation is known; how a shape is presented is the runner's job). `Eq[AssertionError]` is structural —
  same shape, same values — so `expect` can check for a specific failure without coupling tests to presentation;
  `Show[AssertionError]` is a compact one-line debug rendering used when an error is itself a compared value.
  **Assertions signal failure by raising `AssertionError` through the `Throw` effect**; a passing assertion
  returns `unit`. This is why a body carries `{Throw[AssertionError]}`. Provides `success` (never raises),
  `fail(reason)`, the infix `actual shouldBe expected` and `actual shouldNotBe unexpected` (over any `Eq &
  Show`, raising `NotEqual` / `UnexpectedlyEqual`), `expect(error, body)` (passes iff `body` raises exactly
  `error`) and the infix `body message newMessage` (`infix left below shouldBe` — rewrites a failing body's
  message).

  **Both `expect` and `message` wrap a body that performs.** Each declares `{Throw[…]}` on its body slot and
  discharges it there; everything else the body does rides outward to the case. `message` works even though it
  raises `Throw[AssertionError]` *itself* — discharge installs a frame, and the nearest enclosing frame is its
  own, so a definition may discharge the very effect it declares. `message` spells its discharge
  `runThrow[AssertionError, Unit](body)`: the actual is a **parameter reference**, so no declaration determines
  the supplied entry's arguments and the call must say which `Throw` it discharges. That rejection is deliberate —
  defaulting would compile a discharge against a frame it will not meet.

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
  `suite: {Writer[List[TestResult]]} Unit`, which supplies the registration effect and nothing else, so a suite's
  own effects are charged to `runSuite`'s `{Console}` row and bound by whoever ran the runner. The suites after it
  arrive as `rest: {} List[List[String]]`, still unrun — the empty row adds nothing to them — so suites run in
  gathered order and each one's own output precedes its own report. Both `rest` and the discharged `Writer` are
  `val`-bound before they meet `++`, whose element parameter is a plain type parameter and so may not receive a
  computation; the fold's seed `noFailures` is therefore plain `List[List[String]]` data.

  The reporting is a pure-then-print split, and deliberately so: **the runner discharges nothing beyond the suite's
  `Writer`** — each body ran at its `in`, so `failureLines` merely folds `result.outcome` into that case's
  **failure lines**, empty exactly when it passed. `report` derives both the counts and the detail body from the one
  list, and `runSubject` is the only thing that prints. That split is also what makes it *compile*: `foldPair`'s
  result parameter declares no effect row, so a `{Console}` computation may not be routed through it (rule 4 —
  a plain type parameter is a payload). Keep the fold's body pure and `.foreach(printLine)` the lines it yields
  — which is why `runSubject` reads its group through the pure `groupSubject`/`groupResults` projections (the
  stdlib's `keyOf` trick) rather than printing inside a `foldPair`. `subjectOf` is the separate grouping key,
  reaching through `result.testCase.subject`.
  `describe` is the one place turning an `AssertionError` shape into presentation lines, and the `ansi*` helpers
  the one place holding an escape sequence. `header` picks its line with `fold`, **not** `if..else`: `else` and
  `++` have no declared relative precedence, so an `if..else` whose arms concatenate strings does not compile.

**The one architectural idea worth internalizing: tests register by name, via compile-time
reflection — there is no central list, no annotations, no import wiring.** The runner calls
`foldNamedValues("testCases", noFailures, runSuite)` (from `eliot.compiler.Reflect`), which reifies *every*
top-level value literally named `testCases` across all modules on the compiler path as a right fold —
`runSuite(name₁, suite₁, runSuite(name₂, suite₂, noFailures))`. **Reflection reifies code, not data**: each
gathered suite is an *argument* of a slot `runSuite` declares, never an element stored in a list, which is
exactly what lets a suite be a computation. (A list element is a payload by rule 4, and that is why the earlier
`namedValues[Test]` shape could not work.) `namedValues` remains, as the same fold at the free monoid, for
gathering plain data.

To add tests, declare `def testCases: Test = { "…" should "…" in { … } … }` in any module inside a compiled
source root, widening the return type to what those tests need — `{Console} Test` to perform console effects for
real. `test/eliot/test/BasicAssertionsTests.els` is the worked example for plain cases, and
`test/eliot/test/example/GreeterTests.els` for the effectful ones. A suite is picked up simply by being on the
path; nothing references it. Suites are folded in qualified-name order, so a run is reproducible. (One
`testCases` per module — the reflection gathers one value per module under that name, the way `PluginRegistry`
gathers `contribution`.)

> **Watch the build cache.** The compiler caches facts in the output directory (`target/.eliot-*`), and a
> `NamedValuesIndex` was once observed surviving a module's addition, so a newly added suite silently did not
> run while the build reported success — tests that do not run look exactly like tests that pass. It has not
> reproduced since; if a suite you just wrote does not appear in the report, `rm target/.eliot-*` and rebuild
> before looking anywhere else.

## Testing effectful code

There are three ways to write a test that involves effects, and all three register into the same suite.

### Real effects, inline — the default

The suite's return type says what its tests may do. Declare the effect and write the body inline; nothing names
an implementation, and the effect is bound by whoever ran the runner:

```eliot
def testCases: {Console} Test = {
   "real effects" should "be available to a body whose suite declares them" in {
      printLine(" | (printed by a test performing a real Console effect)")
      success
   }
}
```

A failed assertion stops **that case** — `in` supplies `Throw[AssertionError]` per case — and the next case
still runs. A suite declaring plain `Test` rejects a body that performs ("This value performs the effect
'Console' but does not declare it"), so opting out of effects is a matter of *not widening the return type*.

### Test doubles — a named implementation, and one `with`

Production code that declares `{Console}` commits to no interpretation: it names no implementation, so whoever
runs it decides. In production that is the synthesized entry point binding the platform's default; in a test it
is one `with`. `test/eliot/test/example/` is the worked example: `Greeter` is the application under test,
`FakeConsole` holds the doubles and the words that run a body against them, and `GreeterTests` registers all
four styles in **one** suite.

Four properties fall out of the design rather than being added for testing:

- **A double is one declaration.** `implement fakeConsole: Console { … }` needs no type to hang on, no
  colocation with the ability, and raises no coherence question: a **named** implementation is never searched,
  so it may freely overlap the platform's default.
- **A double cannot cheat.** A user module declares no natives and the platform's are private to its layer, so
  an implementation reaches the world only through the effects **its own clauses declare**.
- **A double keeps its own state through an effect**, not through a carrier: `fakeConsole`'s clauses declare
  `{State[Session]}`, and that entry is charged where the double is *bound* — the slot of `onConsole` — and
  discharged there by `runStateToValue`. It never appears in the code under test's row.
- **Interpretation is per effect, not per program.** `body with fakeConsole with fakeTranscript` binds two
  doubles and leaves everything else at its default.

**The seam is the slot's type.** An author's discharge word declares the doubles on the slot, so the caller
writes nothing and the case reads as a bare word with a block, exactly like an inline one:

```eliot
def onConsole(body: {Console, Transcript} Unit with fakeConsole with fakeTranscript): {Throw[AssertionError]} Unit =
   runStateToValue(Session(empty, ""), body)
```

`Console` and `Transcript` are entries `onConsole`'s own row does not name, so the slot *supplies* them, bound by
its `with`s; the `State[Session]` the doubles perform is supplied in turn by `runStateToValue`; and the case's
assertions ride outward into `{Throw[AssertionError]}`, exactly where `in` discharges them. So the word answers
the assertion effect and `in` cannot tell it apart from an inline body.

### Asserting part-way through a faked run, and run-then-assert

Both work, and neither needs a helper definition. `Transcript` is an effect the *test* owns, with no counterpart
in production, reading back what the body has written so far — which is what a mid-run assertion asserts on:

```eliot
"farewell" should "say goodbye, then invite a return" in onConsole {
   printLine("--")
   transcript shouldBe "--\n"                          // asserted mid-run, before the rest happens
   farewell("Bob")
   transcript shouldBe "--\nGoodbye, Bob.\nCome back soon!\n"
}
```

`onConsoleReading(input, { … })` is the same word for a body that reads, with `readLine`'s lines scripted.

Run-then-assert is now a *choice* rather than a shape the language forces: there is no carrier to instantiate, so
`transcriptOf(input, program)` may be written anywhere, an effectful body included. Reach for it when the
assertion is about a finished transcript rather than the steps:

```eliot
"greet" should "greet whoever the console offers" in {
   transcriptOf(singleton("Bob"), greet) shouldBe "Hello, Bob!\n"
}
```

### What the carrier model took with it

Everything below used to be a rule of this framework and is now simply absent. Do not reintroduce any of it, and
treat a comment citing one as history:

- **`pure { … }`** — its whole meaning was pinning a body to `Id`. A row says what it *supplies*, not what it
  forbids, so "this body may perform nothing at all" has no spelling; a bare `Test` suite is how a case is kept
  pure now. (Compiler `docs/effects.md` §2.6, §7.1.)
- **The fake carrier**, its `Effect` instance, and the rule that a fake gets no lifting because it has no
  `Suspend` — the n² cross-lift wall, and "do not stack over a fake" with it.
- **The region rule** and the `{| Recorded}` capture tag that opted out of it.
- **Pinned rows** (`{Throw[AssertionError] | Id}`): an effect row has no base, and writing one is a hard error.
- **`message` cannot wrap a body that performs** — it can, because a definition may discharge the effect it
  declares.
- **Run-then-assert as a necessary shape**, and the helper definitions a faked case used to need.

What survives unchanged: `Dep[X]` + `provide` for a seam you want stated in the signature, and swapping the
platform layer as the whole-program integration answer.

### What still does not work

- **A computation may not be an argument of a plain type parameter** — `caseFailures ++ rest` is rejected
  ("This argument is a computation, but argument 2 of '…' declares no effect row"). Bind it to a `val` first, or
  hand it to a slot that declares a row. This is rule 4, and it is the reason reflection folds instead of
  collecting.
- **A discharger's type arguments are sometimes written by hand.** Where no declaration determines a supplied
  entry's arguments — the actual is a parameter reference, or raises nothing, or the slot names the same ability
  twice — the call spells them, as `message` does. The rejection is loud and names the fix.
- **A row alias works in return position only.** `suite: Test` as a parameter is an error; a slot writes its row
  out.
- **`import eliot.collection.List` shadows same-named prelude operations in that file, silently.** Both are
  explicit imports and `List` wins. There is no shadowing diagnostic, so a file whose resolution goes somewhere
  surprising is worth checking for this first.

The framework compiles and runs: `src` + `test` builds `target/Runner.jar`, which prints one `✔`/`✗` line
per test subject with its pass and failure counts, then a closing summary line for the whole run.

> **Compiler version.** Builds on compiler `2c3db71` (2026-09-12) — effects v6 with the **marked binding binder**
> and the **row alias reached by ordinary name resolution**, which is what lets `Test` be declared in
> `eliot.test.Test` and named from a suite in another file. No carrier is named anywhere in this repository.

## Building and running (compiler CLI)

There is no build system in *this* repo. Compilation is driven by a sibling checkout of the Eliot
compiler (see `eliot.paths` for its location), whose `examples.run` Mill task auto-appends the
`lang`/`stdlib`/`jvm` layer source roots. You pass this project's own roots as positional arguments:

```bash
cd ../eliot                              # the compiler checkout
./mill examples.run jvm exe-jar -m eliot.test.Runner \
   ../eliot-test/src \
   ../eliot-test/test \
   -o ../eliot-test/target
java -jar ../eliot-test/target/Runner.jar   # runs the discovered tests
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
the layer roots, and you supply `src`/`test`. It is git-ignored, being machine-local; keep it in sync with
the CLI invocation if you change either.
