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

- `eliot.test.Test` — `data TestCase(name, body)`, where `body: {Throw[AssertionError]} Unit`. A test
  is a name and an effectful body. That effect-row field makes `TestCase` **generic over its effect
  carrier** (the `{…}` sugar lifts an inferable carrier `F[_]` onto the type — it is exactly
  `data TestCase[auto F[_] ~ Throw[AssertionError]](name, body: F[Unit])`). The module also declares the
  alias **`type Test = TestCase[ThrowCarrier[AssertionError, Id]]`**, which *pins* that carrier to the
  pure identity carrier `Id`. Pinning is required: values gathered by reflection must be a single
  concrete monomorphic type, and a top-level `def` whose type has an unconstrained `auto` carrier
  cannot be signature-checked (nothing determines the carrier). Pinning to `Id` also means a test body
  may only **assert** (raise via `Throw`) — it performs no I/O, since `Id` has no `Suspend` instance.
- `eliot.test.Assertion` — `data AssertionError(message)` plus the assertion API. **Assertions signal
  failure by raising `AssertionError` through the `Throw` effect**; a passing assertion returns `unit`.
  This is why a `TestCase` body carries the `{Throw[AssertionError]}` effect row. Provides `success`
  (a body that never fails), `fail(reason)`, `assertTrue(condition, reason)`, and
  `assertEquals(expected, actual, reason)` (over any `Eq`). `assertTrue` is `if(condition, unit) else
  fail(reason)` — `else` (from `eliot.effect.Abort`) discharges the `if`'s `Abort`, leaving just
  `{Throw[AssertionError]}`.
- `eliot.test.Runner` — `def main: {Console} Unit`, the executable entry point. For each collected
  test it discharges the body's `Throw` on `Id` (`runId(runThrow(body(testCase)))` → `Either[AssertionError,
  Unit]`) and prints the test name followed by `PASS` or the failure message.

**The one architectural idea worth internalizing: tests register by name, via compile-time
reflection — there is no central list, no annotations, no import wiring.** The runner calls
`namedValues[Test]("testCases")` (from `eliot.compiler.Reflect`), which reifies *every* top-level
value literally named `testCases`, of type `Test`, across all modules on the compiler path. To add a
test, declare `def testCases: Test = TestCase("...", body)` in any module inside a compiled source
root — `test/eliot/test/BasicAssertionsTests.els` is the worked example. It is picked up simply by
being on the path; nothing references it. (One `testCases` per module — the reflection gathers one
value per module under that name, the way `PluginRegistry` gathers `contribution`.)

The framework compiles and runs: `src` + `test` builds `target/Runner.jar`, which prints each test's
name and its `PASS`/failure line.

> **Requires effect-row-in-`data` support in the compiler** (commit "Effect-row sugar in data fields:
> lift carrier onto the data type"). Older compiler checkouts reject `data TestCase(body:
> {Throw[AssertionError]} Unit)` with "Cannot resolve type / Cannot quote neutral value".

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
