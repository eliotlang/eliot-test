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

  A **suite** is a `Writer` computation that *accumulates* a `List[TestResult]` log, and it declares its own
  effect row: `{Writer[List[TestResult]]} Unit` for a suite whose tests perform nothing, `{Writer[List[TestResult]],
  Console} Unit` for one whose tests may print. It is a `Writer`, not `State`, because a suite only ever *appends*
  cases and never reads back what has been registered — the honest, minimal contract for a collector (`Writer` =
  append-only `State`; the log's monoid is `Combine[List]` = concatenation). **The row is open, so no carrier is
  named and none is pinned**: the carrier is whatever the `Runner` is compiled to, which is what lets a test perform
  real effects without the framework knowing any platform. There is no `Test` alias — an alias may not carry an open
  row — and the row is the suite's own statement of what its tests are allowed to do.

  Two infix combinators build a suite in direct style:
  - `"what" should "acceptance"` → a `TestCase` (`infix none def should`). Naming a case is its own record, so
    `TestResult` nests it rather than repeating its fields.
  - `definition in body` → runs the body and registers its verdict (`infix none below should def in`). A `{ … }`
    block of `in` statements *sequences*: each `tell` contributes a singleton `[TestResult]`, the blocks' logs join
    via `Combine[List]`, so the suite collects **every** case in declaration order.

  **`in` discharges the assertion effect and nothing else.** Its body slot is
  `{Throw[AssertionError] | G} Unit` over the suite's own carrier `G`, so `Throw[AssertionError]` is consumed per
  case — a failing assertion stops that case and no other — while every other effect the body performs rides `G` and
  is therefore governed by the suite's declared row. The body is a *slot*, not stored data, so it is written where it
  stands: no test is ever collected unrun, and nothing here holds a suspended computation.

  Three things then decide what a case may do, and they compose freely in one suite:
  - the **suite's row** — the default; `in { printLine(…); … shouldBe … }` performs for real where the row says
    `{Console}`, written inline with no definition of its own;
  - **`in pure { … }`** — opts a case out of effects entirely, whatever the suite allows (see `pure` below);
  - **`in mocked { … }`** — runs the case against `eliot.test.Mock`'s doubles for the base effects, with no
    fixture of any kind (see "Mocking");
  - **an author's own word** — `in onConsole { … }` runs the body on a carrier the test declares, which is
    what a project's *own* ability still needs (see "Testing effectful code").

- `eliot.test.Assertion` — `data AssertionError = Failed | NotEqual | UnexpectedlyEqual | NoErrorRaised`, a sum
  of *failure shapes* carrying the already-rendered values (assertions `show` at the raise site, where the
  instance is known; how a shape is presented is the runner's job). `Eq[AssertionError]` is structural — same
  shape, same values — so `expect` can check for a specific failure without coupling tests to presentation;
  `Show[AssertionError]` is a compact one-line debug rendering used when an error is itself a compared value.
  **Assertions signal failure by raising `AssertionError` through the `Throw` effect**; a passing assertion
  returns `unit`. This is why a body carries `{Throw[AssertionError]}`. Provides `success` (never raises),
  `fail(reason)`, the infix `actual shouldBe expected` and `actual shouldNotBe unexpected` (over any `Eq &
  Show`, raising `NotEqual` / `UnexpectedlyEqual`), `expect(error, body)` (passes iff `body` raises exactly
  `error`; discharges the body with `runThrow` and folds the `Either`), and the infix `body message newMessage`
  (`infix left below shouldBe` — rewrites a failing body's message). `expect`'s body row is **open**
  (`{Throw[E]} Unit`), so it supplies one `Throw` layer over whatever carrier the case runs on and can wrap a body
  that performs — a body that prints *and* raises is checkable. `message` cannot follow; see "What does not work".

  It also holds `shouldBeTrue`/`shouldBeFalse` (`Bool` has no `Show`, so `shouldBe true` does not compile)
  and the infix `body describedAs newMessage` — named that, not `message`, because `eliot.file.File` exports a
  `message` too and no file may import both.

  It also holds **`pure`** — `def pure(body: {Throw[AssertionError] | Id} Unit): {Throw[AssertionError]} Unit` —
  the word that **forces a case to be pure**. The pin to `Id` is its whole meaning: `Id` has no `Suspend`, so a body
  handed to `pure` can perform nothing, and a `printLine` inside one is a compile error ("The effect 'Console' cannot
  run here, because the computation it runs in is pure…") *even when the enclosing suite declares `{Console}`*. It
  runs the body down to a verdict and reflects that verdict back into the assertion effect with `orRaise`, so `pure`
  stands exactly where an effectful body stands and `in` cannot tell them apart. The actual discharge is
  `outcomeOf(body) = runId(runThrow(body))` — the one place in the framework that names a carrier, and what `expect`
  and `message` are built from too, which is why `runThrow` appears exactly once here.

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

  `runSuite` receives each suite as a computation in a slot it **declares** — `suite: {Writer[List[TestResult]] | G}
  Unit` over its own carrier `G ~ Console & Effect` — and receives the suites after it as `rest: G[List[List[String]]]`,
  still unrun. So suites run in gathered order, each one's own output precedes its own report, and a suite's effects
  ride the carrier the runner was compiled to. Both `rest` and the discharged `Writer` must be `val`-bound before they
  meet `++`, whose element parameter is a plain type parameter and so may not receive a computation; the fold's seed is
  `noFailures`, a `{Console} List[List[String]]` rather than a bare `empty`, because `rest` declares a computation.

  The reporting is a pure-then-print split, and deliberately so: **the runner discharges nothing beyond the suite's
  `Writer`** — each body ran at its `in`, so `failureLines` merely folds `result.outcome` into that case's
  **failure lines**, empty
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
`foldNamedValues("testCases", noFailures, runSuite)` (from `eliot.compiler.Reflect`), which reifies *every*
top-level value literally named `testCases` across all modules on the compiler path as a right fold —
`runSuite(name₁, suite₁, runSuite(name₂, suite₂, noFailures))`. **Reflection reifies code, not data**: each
gathered suite is an *argument* of a slot `runSuite` declares, never an element stored in a list, which is
exactly what lets a suite be a computation. (A list element is a payload by rule 4, and a stored row must be
pinned — which is why the earlier `namedValues[Test]` shape forced the suite to pin its carrier to `Id`, and
why nothing pins one now.) `namedValues` remains, as the same fold at the free monoid, for gathering plain data.

To add tests, declare `def testCases: {Writer[List[TestResult]]} Unit = { "…" should "…" in pure { … } … }` in
any module inside a compiled source root, widening the row to what those tests need — `{Writer[List[TestResult]],
Console} Unit` to perform console effects for real. `test/eliot/test/BasicAssertionsTests.els` is the worked
example for pure cases, and `test/eliot/test/example/GreeterTests.els` for the effectful ones. A suite is picked
up simply by being on the path; nothing references it. Suites are folded in qualified-name order, so a run is
reproducible. (One `testCases` per module — the reflection gathers one value per module under that name, the way
`PluginRegistry` gathers `contribution`.)

> **Watch the build cache.** The compiler caches facts in the output directory (`target/.eliot-*`), and a
> `NamedValuesIndex` was once observed surviving a module's addition, so a newly added suite silently did not
> run while the build reported success — tests that do not run look exactly like tests that pass. It has not
> reproduced since; if a suite you just wrote does not appear in the report, `rm target/.eliot-*` and rebuild
> before looking anywhere else.

- `eliot.test.Mock` — **the doubles, and the vocabulary a test writes.** One carrier (`Mock`), one
  `Effect` instance, one generic `implement[E ~ Show] Throw[E, Mock]` that serves *every* failure channel
  including a project's own error types and `AssertionError` itself, and a concrete double for every base
  effect ability: `Console`, `Process`, `FileSystem`, `Environment`, `Log`. A test writes **no fixture** —
  it arranges, acts and asserts in one `in mocked { … }` block.

  Arranging is an effect (`ability Mocking`), which is why `mocked` takes only a block: `whenSpawning`
  (with `succeeding`/`failing`/`exiting`), `whenSpawningCreates`, `whenReading`, `withFile`,
  `withDirectory`, `withVariable`, `withArguments`, `withWorkingDirectory`. **The most recent arrangement
  wins**, so a body may act, re-arm and act again, and a shared arrangement is an ordinary `{Mocking} Unit`
  definition — this framework's `@Before`, with no annotation behind it.

  Verifying is an effect too (`ability Calls`): `calls`, `callsMatching`, `lastCall`, `callCount`,
  `wasCalled`, `wasNeverCalled`, `wasCalledOnce`, `wasCalledTimes`, `wasCalledAtLeast`, `wasCalledAtMost`,
  `wereCalledInOrder`, `nothingWasCalled`, `onlyTheseWereCalled`, `forgetCalls`. Matching is by
  **containment** of the recorded line. `raising(report, { … })` expects a failure — `expect` cannot, since
  it stacks a `ThrowCarrier` over the carrier and a double earns no lift through it.

  **Why the framework owns this and a project cannot**: an instance must live with its ability or with a
  type argument, and an ability may have at most one carrier-generic instance — so a double is necessarily
  concrete and necessarily colocated with the carrier. `docs/mocking.md` has the measurements. What a
  project still writes is a carrier for *its own* abilities, which is what `eliot-build`'s `TablePackages`
  is.

## Testing effectful code

There are three ways to write a test that involves effects, and all three register into the same suite.

### Real effects, inline — the default

A suite's row says what its tests may do. Declare the effect and write the body inline; no carrier is named, no
definition of its own is needed, and the effect runs on whatever carrier the `Runner` was compiled to:

```eliot
def testCases: {Writer[List[TestResult]], Console} Unit = {
   "real effects" should "be available to a body whose suite declares them" in {
      printLine(" | (printed by a test performing a real Console effect)")
      success
   }
}
```

A failed assertion stops **that case** — `in` discharges `Throw[AssertionError]` per case — and the next case
still runs. A suite that declares only `{Writer[List[TestResult]]}` rejects a body that performs ("This value
performs the effect 'Console' but does not declare it"), and `in pure { … }` rejects one *whatever* the suite
declares, so opting out of effects is always available and always enforced.

### A faked carrier, for determinism

Real effects are not always what a test wants. Production code that declares `{Console}` can equally be run
against a test double, because **the carrier is the injection point** (the compiler's own
the compiler's `docs/effects.md` §6, and `examples/src/EffectsTestFramework.els`). The test declares its own pure carrier
and its own instance of the ability for it; production code is untouched and names no carrier.
`test/eliot/test/example/` is the worked example: `Greeter` is the application under test, `FakeConsole` is the
carrier and its discharge words, and `GreeterTests` registers all four styles in **one** suite.

A fake carrier cannot cheat: it has no `Suspend` instance, and `Suspend` is the only route to a native side
effect, so a body running on it is structurally incapable of touching the real console.

### Run-then-assert

A fake run inside a `pure { … }` body would be written at that body's own carrier, so the run goes in a plain
`def` beside it and the body asserts on the value that answers. (This style is a *choice*, not a requirement —
since the capture tag a run may be written inline; run-then-assert is still the clearer shape when the assertion
is about a finished transcript rather than the steps.)

```eliot
private def greetTranscript: String = transcriptOf(singleton("Bob"), greet)   // fake run: its own definition

"greet" should "greet whoever the console offers" in pure {
   greetTranscript shouldBe "Hello, Bob!\n"                                   // assertion: the framework's body
}
```

### Direct style, asserting part-way through a faked run

This **does** work, and the compiler's `docs/effects.md` §7.7 has been corrected to say so — but not the way its
retired L3 note tried it. Pinning the assertion effect *over* the fake carrier (`{Throw[AssertionError] | Session}
Unit`) does fail, for the two reasons L3 recorded: a fake's abilities have no canonical carrier, so they cannot be
pinned-row entries,
and the ability would need an instance for the whole stack rather than the base. The move that works is to stop
stacking: give the **fake carrier itself** a `Throw[AssertionError]` instance, so assertions ride the same
carrier as the faked effects. No row is pinned, so no cross-lift is needed:

```eliot
data Recorded[A](runRecorded: Session => Pair[Either[AssertionError, A], Session])

implement Console[Recorded] { … }                       // the faked effect
implement Throw[AssertionError, Recorded] {             // …and assertions, on the same carrier
   def raise[A](err: AssertionError): Recorded[A] = Recorded(s -> Pair(Left(err), s))
}

def onConsole(input: List[String], body: Recorded[Unit]): Outcome =   // the author's own discharge word
   first(runRecorded(body)(Session(input, "")))
```

`Effect[Recorded]`'s `flatMap` short-circuits on a `Left`, so a failed assertion stops the rest of the body the
way a real failure stops a real test. The body then reads as an ordinary script:

An author's discharge word answers the **assertion effect** (`{Throw[AssertionError]} Unit`), exactly as `pure` does,
so `in` cannot tell the two apart. Give it a single parameter and the case reads as a bare word with a block, the
same shape as `pure`: `"…" should "…" in onConsole { … }`. It runs the body down to an `Outcome` and reflects that
back with `orRaise`. `onConsoleReading(input, { … })` is the same word for a body that reads, with `readLine`'s
lines scripted. So all three ways to give a case a body read alike — inline on the suite's row, `pure { … }`, and
the author's own word:

```eliot
private def farewellOutcome: Outcome = onConsole(empty, farewellScript)

private def farewellScript: {Console, Transcript, Throw[AssertionError]} Unit = {
   printLine("--")
   transcript shouldBe "--\n"                          // asserted mid-run, before the rest happens
   farewell("Bob")
   transcript shouldBe "--\nGoodbye, Bob.\nCome back soon!\n"
}
```

**A faked case now costs no helper definitions at all**, because `onConsole` declares its body slot with the
**capture tag** `{| Recorded} Unit` — the pinned row at zero entries, which is the same type as `Recorded[Unit]`
but declares that the slot *hosts a computation on that carrier* (compiler `docs/effects.md` §2.3, shipped as W3
on 2026-09-04). The tag makes the slot a capture, so the elaborator writes nothing into it and the checker
instantiates the body at `Recorded` — even at the `in` site, inside the suite's own region. Both the run and a
multi-statement body are written inline.

Without the tag the **region rule** still governs, and it is worth knowing why the two definitions used to be
forced: a foreign concrete carrier can only be instantiated in a region with no ambient carrier of its own, and
the suite is a region, so an untagged fake run written there is written at the suite's own `Writer` stack. That
is what `transcriptOf`'s and `onConsole`'s tags now opt out of.

### The one rule to internalize about fakes: do not stack over them

A real effect instance is **carrier-polymorphic** (`implement[F[_] ~ Suspend] Console[F]`), so it applies at any
stack whose base can suspend — `Suspend` is doing the work an mtl `lift` would, which is why real effects appear
to compose freely. A fake instance is **monomorphic at one concrete carrier** (`implement Console[Recorded]`) —
which is exactly what makes it uncheatable — and therefore gets **no lifting at all**. The moment a stdlib
control carrier is stacked over a fake, resolution fails at the stack:

```
No ability implementation found for ability 'Transcript' with type arguments [{Throw[AssertionError] | Recorded}]
```

So give the fake carrier its own instance of *everything* the body needs, assertions included
(`implement Throw[AssertionError, Recorded]`), and let it all ride one carrier. That is why `FakeConsole`
looks the way it does, and it is what makes the interleaved style work. A missing cell can be hand-written
(`implement[E, G[_] ~ Transcript & Effect] Transcript[ThrowCarrier[E, G]]` does resolve and run) but it is one
instance per (ability × carrier layer), so reach for it only when the no-stacking answer genuinely fails.

### What does not work

- **`Throw` composes with no other stdlib control effect over `Id`.** The cross-lift matrix has no
  `Throw[E, StateCarrier[…]]`, `State[S, ThrowCarrier[…]]`, `Throw[E, DepCarrier[…]]` or
  `Dep[X, ThrowCarrier[…]]`, so a discharge word over a pinned stack (`{Throw[AssertionError], State[S] | Id}`,
  either pin order) does not compile. This is the same n² gap as the fake-lifting one above, and the same
  single-carrier answer applies — which is why the faked route goes through one custom carrier rather than a
  stack, and serves `State`/`Dep` fakes too.
- **`message` pins its body to `| Id`, and cannot be un-pinned.** Not a choice: a parameter row entry is
  *supplied* — stacked as an extra layer — only when the definition's own declared return row does not already
  name it. `message` raises `Throw[AssertionError]` itself, so a `{Throw[AssertionError]}` parameter row would
  denote the same carrier rather than one above it. `expect` **is** un-pinned (its `E` is its own binder, a
  distinct entry), so it wraps bodies that perform; `message` still cannot. Neither can wrap an assertion that
  reads a fake, for the stacking reason above.
- **`import eliot.collection.List` shadows `Effect`'s `map`/`flatMap` in the same file, silently.** Both are
  explicit imports and `List` wins, so carrier code in that file resolves to the list combinator and dies with a
  message pointing nowhere near the cause — `No ability implementation found for ability 'X' with type arguments
  [List]`. There is no shadowing diagnostic. A file that writes an `Effect`/carrier instance body must therefore
  not import `eliot.collection.List`; put the instance in its own module, or avoid `map`/`flatMap` there.
- **A computation may not be an argument of a plain type parameter** — `caseFailures ++ rest` and
  `append(list, suite)` are both rejected ("This argument is a computation, but argument 2 of '…' declares no
  effect row"). Bind it to a `val` first, or hand it to a slot that declares a row. This is rule 4, and it is the
  reason reflection folds instead of collecting.

The framework compiles and runs: `src` + `test` builds `target/Runner.jar`, which prints one `✔`/`✗` line
per test subject with its pass and failure counts, then a closing summary line for the whole run.

> **Compiler version.** Builds on compiler `c1bc4704` (2026-09-03). Needs **`foldNamedValues`** (shipped 2026-08-18 —
> "Reflection reifies code, not data"), which is what allows a suite to be a computation and so removed every
> pinned carrier from the framework: gathered values are handed to a slot the algebra declares instead of being
> stored in a `List`. It also needs the ambient **`Writer` effect** + **`Combine[List]` monoid** (shipped
> 2026-07-21 — `Writer` is `State` restricted to append-only, `Combine` gained `empty`). No `data` here stores an
> effect row, no alias carries one, and the only carrier named anywhere in the framework is the `Id` inside
> `outcomeOf`, which is what makes `pure` mean what it says.

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
