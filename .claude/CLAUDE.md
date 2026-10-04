# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`eliot.test` — a unit-testing framework for the [Eliot language](https://github.com/eliotlang/eliot),
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

A file is a module: `src/eliot/test/Test.els` is module `eliot.test.Test`. `README.md` is the user's guide; this
section is how the pieces fit. Six modules, each owning one concern and keeping its representation private:

- `eliot.test.Test` — the data and the DSL. `data TestCase(subject, shouldPhrase)` names a case;
  `data Outcome = Passed | FailedWith(assertionError)` is what running it came to; `data TestResult(testCase,
  outcome)` the two together, with `passed(result)` the one predicate everything else asks. **Nothing stores a
  computation**: a body runs where it is written. `type Test = {Writer[List[TestResult]]} Unit` is a **row alias**
  — what a suite returns, composing with a written-out row (`{Console} Test`) to widen what its cases may do. It
  works in return position only, which is why `in`'s and `runSuite`'s slots write the row out. `"…" should "…"`
  builds a `TestCase`; `testCase in body` runs `body`, discharging `Throw[AssertionError]` — the one entry its
  slot supplies — and `tell`s the result, so a failing assertion stops that case and no other.

- `eliot.test.Assertion` — `data AssertionError = Failed | NotEqual | UnexpectedlyEqual | NoErrorRaised |
  Described`, kinds of failure carrying already-rendered values (rendering happens at the raise site, where the
  `Show` is known; presenting is `Report`'s job). `Eq` is structural, `Show` one labelled line. The words:
  `success`, `fail`, `shouldBe`/`shouldNotBe` (any `Eq & Show`), `shouldBeTrue`/`shouldBeFalse` (`Bool` has no
  `Show`), `describedAs` (keeps the original failure as `Described`'s `originalFailure`, rendered — a recursive
  field could not be consumed without recursion), `expect` (the exact error) and `raising` (part of its text —
  the two siblings live together here; `raising` used to sit in `Mock` though it needs no double).
  `describedAs` is not named `message` because `eliot.file.File` exports one, and for the same reason no
  constructor field here may be called `message`.

- `eliot.test.Report` — **pure**: everything about the report, testable without running a suite.
  `data Style = Colored | Plain | Teamcity`, `data Tally(passedCount, failedCount)`, `tally`, `allPassed`,
  `suiteReport(style, suiteName, results)` (the module name, then one line per subject in first-appearance order
  with each failure's lines under it) and `summary(style, tally)`. `describe` is the one place an
  `AssertionError` becomes lines, `painted` the one place holding an escape sequence. The colour helpers are not
  called `success`/`failure`, which `Assertion` exports. **`Teamcity` is the same report as service messages** for an
  IDE's test runner to read: the module is a suite, each subject a suite inside it, each case a test named by its
  `should` phrase; a failed case is `testFailed` (with `expected`/`actual` where it is a `NotEqual`, which is what lets
  an IDE offer a diff) and is still `testFinished`. `escaped` is the one place the protocol's reserved characters
  (`|` first, then `'`, newline, `[`, `]`) are handled. It has no colour and `summary` stays plain text. What a case
  prints itself comes out *before* its suite's messages — the report is printed after the suite has run — so an IDE
  shows it in the console but cannot attach it to the test.

- `eliot.test.Arguments` — **pure**, like `Report`: what the runner's command line (`Environment.arguments`) means.
  `namesIn` (arguments not starting `--`), `selects(names, suiteName)` (module equal to a name or below it, by
  whole parts — the dot is part of the match; no names selects all), `styleOf(arguments, fallback)` (`--format=plain|
  colored`, first wins), `unknownOptions`, and `nothingMatched(names, tally)`. The runner **refuses rather than
  passes**: an unknown option exits 2 before running, and names that select no case exit 2 instead of "all clear"
  — a silently ignored filter looks like a pass. Operators `+`, `==` and `&&` have no relative precedence, so
  these expressions parenthesize. `--format=teamcity` selects the service-message report (see `Report`).

- `eliot.test.Runner` — `def main: {Console, Process, Environment} Unit`, and nothing but effects:
  `foldNamedValues("testCases", noResults, runSuite)` folds every suite; `runSuite` runs one
  (`runWriterToLog(suite)`), prints its `suiteReport`, **then** forces `rest`, so each suite's report sits next to
  what its own cases printed; `main` prints the `summary` and exits 1 unless `allPassed`. `NO_COLOR` selects
  `Plain`, and `--format=` wins over it. Suites run under `runSuite`'s `{Console, Environment}`; `Process` is
  `main`'s alone. **`runSuite` reads `arguments` itself** to skip an unselected suite (forcing only `rest`): the
  fold's `combine` must be a declared value, never a lambda, so a filter cannot be handed to it as a parameter.

- `eliot.test.Mock` — the doubles and the words a test writes (README, "Mocking"). Public: `mocked`, the
  arranging and verifying words, the effects `Mocking` and `Calls` (a project names them in rows — eliot-build's
  `TableGit` declares `Mocking`), and the five named implementations. **Private**: everything they keep —
  `Recording(journal, arrangements, fileTree, pendingInput)` in `State[Recording]`, which `mocked` supplies and
  discharges; `Call = Asked(operationName, operationArguments) | Spawned(spawnDirectory, commandLine)`; `Arrangement` (answers:
  spawns, creations, variables, arguments, working directory); and `Entry = FileEntry | DirectoryEntry`, the
  in-memory file tree, kept apart from the arrangements. Two rules carry the design:
  - **Matching is by words**, in one function, `matches`: a fragment matches when its words appear in the call's
    description next to each other and in order (both sides space-padded, then `contains`). `whenSpawning` and
    every verification use it, against the same description — `printLine hello`, `/work$ git fetch`.
  - **Every double's operation is `recorded(update, answer)`**: change the recording, answer from the one before
    the call. `answering`, `changingFiles` and `noting` are its three shapes.
  The tree implies directories: a directory exists when something is inside it, `listDirectory` answers the first
  step down from each path inside, `walk` files only, `delete` the path and everything under it, and paths are
  keyed without a trailing `/`. **The doubles cannot raise `IoError`**: the base's `type IoError` has no
  constructor a user module can call, so a missing file reads `""`; making failures arrangeable needs one added
  to eliot's base first.

**Tests register by name, via compile-time reflection — there is no central list.** `foldNamedValues` reifies
every top-level value named `testCases` as a right fold, `runSuite(name₁, suite₁, runSuite(name₂, suite₂,
noResults))`, each suite an *argument* of a slot `runSuite` declares — which is what lets a suite be a
computation. `name` is the qualified name, `pkg.Module::testCases`. Suites run in qualified-name order. One
`testCases` per module.

> **Watch the build cache.** A `NamedValuesIndex` was once observed surviving a module's addition, so a new suite
> silently did not run. If a suite does not appear in the report, `rm -rf target/.eliot-*` and rebuild.

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
`test/src/eliot/test/example/` is the worked example of a project's side of that: `Greeter` is the application under
test and `GreeterTests` registers its pure, mocked and real-effect cases in **one** suite. The `Terminal` sketch
below is illustrative — for a *base* effect the doubles are already written (see `eliot.test.Mock`), so nothing in this
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
  (`catch[AssertionError, Unit](…)` in `describedAs`, `runThrow[E, Unit](body)` in `raising`), the actual
  **raises nothing** (`raising[AssertionError]("expected", printLine(…))` in `MockVerificationTests`), or the
  slot's row names the **same ability twice** (`catch[IoError, Unit](…)` in `mocked`). A bare `raise("…")` is the
  generic `raise[A](err: E)`, so its declaration says nothing either: `raising[String]("…", raise("…"))` in
  `BasicAssertionsTests`.
- **A computation may not be `val`-bound or dot-chained before being discharged.** Both are rowless positions,
  so the call *runs* there and the enclosing def is charged with the effect. Pass it to the discharger directly.
- **`import eliot.collection.List` shadows `Effect`'s `map`/`flatMap`** — no longer a hazard here, since there
  is no `eliot.carrier` and nothing to shadow.
- **A stored computation's binding is fixed where it is constructed.** A `with` applied to it later is an
  error, not a rebinding.
- **A constructor passed as a function value dies at run time** in eliot `v0.7` (`NoSuchMethodError` on
  `eliot.test.Test.FailedWith()`), so `Test.outcomeOf` writes `error -> FailedWith(error)` rather than
  `FailedWith`.
- **A field accessor is a module-level name.** `data X(arguments: …)` in a module importing
  `eliot.system.Environment` collides with its `arguments`, and a helper may not share a field's name — which is
  why `Mock`'s fields carry longer names than its parameters.

`./eliotw test` prints, per suite, its module name and one `✔`/`✗` line per subject with each failure detailed
underneath, then one summary line for the whole run. **120 cases, all passing.**

> **Compiler version.** Needs eliot `v0.7`: the base's `when`/`unless`, `someIf`, `filterMap`/`findMap`,
> `includes`, `Eq[Option]` and `first`/`second`, which the assertions and the doubles are written with, and the
> fix for two `if..else`s over two kinds of `Option` sharing one `runAbort` (`NoSuchMethodError`). Before that,
> `2c3db71` (2026-09-12) — effects v6, the fix for an under-applied ability-implementation native (`32406522`,
> which `MockFileSystemTests`' `listDirectory(…).map(show)` hits), and the **row alias reached by ordinary name
> resolution**, which is what lets `Test` be declared in `eliot.test.Test` and named from a suite in another file.

## Building and running

`./eliotw test` is the way in, and it needs nothing installed: the committed wrapper reads the `launcher <tag>`
line of `eliot.pkg`, fetches that launcher once into `~/.cache/eliot/launcher/<tag>/` and runs it. The launcher has
no verbs — the command line is the package name:

```bash
./eliotw test              # compiles and runs this repository's own suites; exits 1 if any case failed
NO_COLOR=1 ./eliotw test   # the same, without colours
./eliotw runner            # target/Runner.jar with src only, no suites in it, not run
```

**Four packages.** `root` (at `.`) is the framework — `src/` and nothing that runs, no platform. **`suite`** is
`root` plus `compiler run -m eliot.test.Runner`: a consumer's test package deps `eliot-test//suite` beside its
platform and writes no line of its own. `runner` builds the same entry point as a jar, and `test` is this
repository's suite, depping `//suite` like everybody's. The line is not on `root`, because every closure holding a
package runs its line and `runner` holds `root`.

The floor is eliot `v0.7`, the first whose base has the conveniences the framework is written with
(`when`/`unless`, `someIf`, `filterMap`/`findMap`, `includes`, `Eq[Option]`, `first`/`second`).
`./eliotw --project-model` (launcher `v0.6`+) prints the resolved roots as JSON when a build picks something
unexpected. `rm -rf target` starts over; `ELIOT_CACHE` moves the launcher cache and `ELIOT_LAUNCHER_REPOSITORY`
points the wrapper at a mirror.

### The compiler CLI, for a change to the compiler itself

The wrapper runs a *published* toolchain, so a change in the compiler checkout is invisible to it. Drive the
compiler directly from a sibling checkout (`/home/robert/personal/eliot`), whose `examples.run` Mill task appends
the `lang`/`stdlib`/`jvm` layer roots; pass this project's roots positionally:

```bash
cd /home/robert/personal/eliot
./mill examples.run run -m eliot.test.Runner \
   /home/robert/personal/eliot-test/src \
   /home/robert/personal/eliot-test/test/src \
   -o /home/robert/personal/eliot-test/target
```

`-m <module>` must come **immediately after the mode** and be fully qualified; `-o <dir>` trails at the end.
Every root that should contribute suites must be passed.

The IDE asks `./eliotw --project-model` for the roots; `eliot.paths` is gone.
