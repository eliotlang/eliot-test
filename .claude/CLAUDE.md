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

## Architecture

A file is a module: `src/eliot/test/Test.els` is module `eliot.test.Test`. The whole framework is
three tiny modules plus a self-test:

- `eliot.test.Test` — the DSL, and **plain data throughout**. A **test case** is `data TestCase(subject:
  String, shouldPhrase: String)`: what it is about and what it should do, with *no body*. What running a
  body answered is `type Outcome = Either[AssertionError, Unit]`, and the two together are `data
  TestResult(testCase: TestCase, outcome: Outcome)`. **Nothing here stores a computation, so nothing here
  pins a carrier** — the framework holds verdicts, not suspended work.

  A **suite** is `type Test = {Writer[List[TestResult]] | Id} Unit` — a `Writer` computation that
  *accumulates* a `List[TestResult]` log (over `Id`). This is the type reflection gathers. It is a
  `Writer`, not `State`, because a suite only ever *appends* cases and never reads back what has been
  registered — the honest, minimal contract for a collector (`Writer` = append-only `State`; the log's
  monoid is `Combine[List]` = concatenation). It is pinned to `Id` for two reasons at once: a gathered type
  must be concrete, and `Id` has no `Suspend`, so a suite can perform nothing beyond its own registration.

  Two infix combinators and a **discharge word** build a suite in direct style:
  - `"what" should "acceptance"` → a `TestCase` (`infix none def should`). Naming a case is now its own
    record, so `TestResult` nests it rather than repeating its fields — the accessor collision that once
    ruled a separate definition type out no longer arises.
  - `definition in outcome` → `tell(singleton(TestResult(...)))` (`infix none below should def in`),
    registering one case into the enclosing suite's Writer log. A `{ … }` block of `in` statements
    *sequences*: each `tell` contributes a singleton `[TestResult]`, the blocks' logs join via
    `Combine[List]`, so the suite collects **every** case in declaration order. **`in` takes plain data**,
    so it names no effect and no carrier.
  - `in pure { … }` — the word between `in` and the block is what runs the body. No new syntax was needed:
    juxtaposition binds tighter than any infix operator, so this parses as `in(testCase, pure({ … }))`.
    Because the outcome is an *argument*, **the body has already run by the time the case is registered** —
    tests run where they are written and only verdicts are collected. A word taking arguments works the same
    way by currying: `in myCarrier(input) { … }`.

  `in` **returns `Test`**; the reflected `testCases` value still **spells its pinned row out** (see reflection
  below). A closed-row alias is carried in only one of the two positions that matter: as of compiler `eb163b1` it
  is accepted in a definition's return position — which is why `in` names the alias again — but rejected where a
  *reflected* value declares it (`namedValues[Test]` over `def testCases: Test` ⤳ "This argument is a
  computation, but argument 2 of 'append' declares no effect row"). So `testCases` writes its row out; once the
  compiler carries the alias at a reflected value too, it can simply become `Test` as well.
- `eliot.test.Assertion` — `data AssertionError = Failed | NotEqual | UnexpectedlyEqual | NoErrorRaised`, a sum
  of *failure shapes* carrying the already-rendered values (assertions `show` at the raise site, where the
  instance is known; how a shape is presented is the runner's job). `Eq[AssertionError]` is structural — same
  shape, same values — so `expect` can check for a specific failure without coupling tests to presentation;
  `Show[AssertionError]` is a compact one-line debug rendering used when an error is itself a compared value.
  **Assertions signal failure by raising `AssertionError` through the `Throw` effect**; a passing assertion
  returns `unit`. This is why a body carries `{Throw[AssertionError]}`. Provides `success` (never raises),
  `fail(reason)`, the infix `actual shouldBe expected` and `actual shouldNotBe unexpected` (over any `Eq &
  Show`, raising `NotEqual` / `UnexpectedlyEqual`), `expect(error, body)` (passes iff `body` raises exactly
  `error`; runs `pure(body)` → `Either` and folds it), and the infix `body message newMessage`
  (`infix left below shouldBe` — rewrites a failing body's message).

  It also holds **`pure`** — `def pure[E, A](body: {Throw[E] | Id} A): Either[E, A] = runId(runThrow(body))`
  — the framework's discharge word and **the one place in it that names a carrier**. The pin to `Id` is the
  word's meaning, not an implementation detail: `Id` has no `Suspend`, so a body handed to `pure` may assert
  and nothing else, and a `printLine` in a unit test is a compile error ("The effect 'Console' cannot run
  here, because the computation it runs in is pure…"). Running the body down to a verdict is the consequence
  of that constraint. `expect` and `message` go through `pure` for the same reason, which is why `runThrow`
  appears exactly once in the framework.
- `eliot.test.Runner` — `def main: {Console} Unit`, the executable entry point.
  `namedValues[Test]("testCases").flatMap(runSuite)` gathers every suite (see reflection below); `runSuite`
  discharges the Writer to its accumulated list (`suite.runWriterToLog.runId`) and **groups it by `subject`**
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

  The reporting is a pure-then-print split, and deliberately so: **the runner discharges nothing** — the body
  ran at its `in`, so `failureLines` merely folds `result.outcome` into that case's **failure lines**, empty
  exactly when it passed. `report` derives both the counts and the detail body from the one list, and
  `runSubject` is the only thing that prints. That split is also what makes it *compile*: `foldPair`'s
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
`namedValues[Test]("testCases")` (from `eliot.compiler.Reflect`), which reifies *every* top-level
value literally named `testCases`, of type `Test`, across all modules on the compiler path. To add
tests, declare `def testCases: {Writer[List[TestResult]] | Id} Unit = { "…" should "…" in pure { … } … }`
in any module inside a compiled source root — the type is `Test` spelled out, because a reflected value
may not declare the closed-row alias as of `eb163b1`, even though `in` itself now returns the alias (see `in` above);
`test/eliot/test/BasicAssertionsTests.els` is the worked example. It is picked up simply
by being on the path; nothing references it. (One `testCases` per module — the reflection gathers one
value per module under that name, the way `PluginRegistry` gathers `contribution`.)

## Testing effectful code

A `pure` body cannot perform I/O, which is the point — but production code that declares `{Console}` (or any
effect) is still testable, because **the carrier is the injection point** (the compiler's own
`docs/testing-effects.md`, and `examples/src/EffectsTestFramework.els`). The test declares its own pure
carrier and its own instance of the ability for it; production code is untouched and names no carrier.

The shape is **run-then-assert**, and the ordering is forced: the fake run must sit in a definition with *no
ambient carrier of its own*, because a region writes every carrier-generic callee at its own carrier. So the
run cannot go inside the `pure { … }` body — it goes in a plain `def` beside it, and the body asserts on the
value that answers:

```eliot
def greetTranscript: String = transcriptOf(greet("Bob"))   // fake run: its own definition

"greet" should "greet the name it was given" in pure {
   greetTranscript shouldBe "Hello, Bob!;"                 // assertion: the framework's body
}
```

Asserting *part-way through* a faked run is not possible: pinning the assertion effect over the fake carrier
(`{Throw[AssertionError] | Session} Unit`) fails because a fake's abilities have no canonical carrier to be
row entries, and because the ability would need an instance for the whole stack rather than the base. That is
L3 in `docs/testing-effects.md`, whose proposed `{| Session}` capture tag would lift it; it is a convenience,
not a prerequisite.

The framework compiles and runs: `src` + `test` builds `target/Runner.jar`, which prints one `✔`/`✗` line
per test subject with its pass and failure counts, then a closing summary line for the whole run.

> **Compiler version.** Builds on compiler `eb163b1`. Needs the ambient **`Writer` effect** +
> **`Combine[List]` monoid** (shipped 2026-07-21 — `Writer` is `State` restricted to append-only, `Combine`
> gained `empty`). It no longer needs pinned effect-row `data` fields at all — no `data` here stores a row.
> **`eb163b1` carries closed-row aliases at a definition's return** — where `f7a546b` rejected them — so `in`
> returns `Test`, which is why the framework now needs `eb163b1` and no longer builds on `f7a546b`. The same
> commit **regressed the alias at reflected values**: it rejects `Test` where a reflected value declares it, so
> the `testCases` value alone spells its pinned row out (`{Writer[List[TestResult]] | Id} Unit`, see `in`
> above). That regression is a compiler bug, and the spelled-out row is the minimal way to keep building across it.

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
