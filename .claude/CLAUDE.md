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

- `eliot.test.Test` — the DSL. A **test case** is `data TestCase(name: String, body: {Throw[AssertionError]
  | Id} Unit)`: a name plus a body that raises an `AssertionError` if it fails. That body field is a
  **pinned effect row** — the concrete `ThrowCarrier[AssertionError, Id, Unit]` stack over the pure
  identity base `Id` (stored `data`-field rows must be pinned). Pinning to `Id` means a body may only
  **assert** (raise via `Throw`), never do I/O, since `Id` has no `Suspend` instance.

  A **suite** is `type Test = {Writer[List[TestCase]] | Id} Unit` — a `Writer` computation that
  *accumulates* a `List[TestCase]` log (over `Id`). This is the type reflection gathers. It is a
  `Writer`, not `State`, because a suite only ever *appends* cases and never reads back what has been
  registered — the honest, minimal contract for a collector (`Writer` = append-only `State`; the log's
  monoid is `Combine[List]` = concatenation).

  Two infix combinators build a suite in direct style:
  - `"what" should "acceptance"` → a `TestCaseDefinition` (`infix none def should`).
  - `definition in { body }` → `tell(append(empty, TestCase(...)))` (`infix none below should def in`),
    registering one case into the enclosing suite's Writer log. A `{ … }` block of `in` statements
    *sequences*: each `tell` contributes a singleton `[TestCase]`, the blocks' logs join via
    `Combine[List]`, so the suite collects **every** case in declaration order.
- `eliot.test.Assertion` — `data AssertionError(errorMessage: String)` (with `Eq[AssertionError]` = equal
  by message, so `expect` can check for a specific failure) plus the assertion API. **Assertions signal
  failure by raising `AssertionError` through the `Throw` effect**; a passing assertion returns `unit`.
  This is why a body carries `{Throw[AssertionError]}`. Provides `success` (never raises),
  `fail(reason)`, the infix `actual shouldBe expected` (over any `Eq` — `if(actual == expected) unit else
  fail(...)`), `expect(error, body)` (passes iff `body` raises exactly `error`; runs `body.runThrow.runId`
  → `Either` and folds it), and the infix `body message newMessage` (`infix left below shouldBe` — rewrites
  a failing body's message).
- `eliot.test.Runner` — `def main: {Console} Unit`, the executable entry point.
  `namedValues[Test]("testCases").foreach(runSuite)` gathers every suite (see reflection below); `runSuite`
  discharges the Writer to its accumulated list (`suite.runWriterToLog.runId`) and **groups it by `subject`**
  (`List.groupBy`, so subjects appear in order of first mention). **One line is printed per subject, not per
  case**: green `✔ subject · N passed` while everything passed, red `✗ subject · N passed, M failed` as soon as
  anything did not, with each failure's `should` phrase and detail lines indented underneath.

  The reporting is a pure-then-print split, and deliberately so: `runTestCase` discharges the body's `Throw` on
  `Id` (`testCase.body.runThrow.runId` → `Either[AssertionError, Unit]`) and answers that case's **failure
  lines** — empty exactly when it passed — so `report` derives both the counts and the detail body from the one
  list, and `runSubject` is the only thing that prints. That split is also what makes it *compile*: `foldPair`'s
  result parameter declares no effect row, so a `{Console}` computation may not be routed through it (rule 4 —
  a plain type parameter is a payload). Keep the fold's body pure and `.foreach(printLine)` the lines it yields.

**The one architectural idea worth internalizing: tests register by name, via compile-time
reflection — there is no central list, no annotations, no import wiring.** The runner calls
`namedValues[Test]("testCases")` (from `eliot.compiler.Reflect`), which reifies *every* top-level
value literally named `testCases`, of type `Test`, across all modules on the compiler path. To add
tests, declare `def testCases: Test = { "…" should "…" in { … } … }` in any module inside a compiled
source root — `test/eliot/test/BasicAssertionsTests.els` is the worked example. It is picked up simply
by being on the path; nothing references it. (One `testCases` per module — the reflection gathers one
value per module under that name, the way `PluginRegistry` gathers `contribution`.)

The framework compiles and runs: `src` + `test` builds `target/Runner.jar`, which prints one `✔`/`✗` line
per test subject with its pass and failure counts.

> **Requires a recent compiler.** Needs pinned effect-row `data` fields (`data TestCase(body:
> {Throw[AssertionError] | Id} Unit)`; older checkouts reject it with "Cannot resolve type / Cannot
> quote neutral value") and the ambient **`Writer` effect** + **`Combine[List]` monoid** (shipped in the
> compiler 2026-07-21 — `Writer` is `State` restricted to append-only, `Combine` gained `empty`).

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
