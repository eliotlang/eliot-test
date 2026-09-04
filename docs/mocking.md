# Mocking Belongs to the Framework: a Plan

Status: **PLAN**, 2026-09-04. Nothing here is built. The four language facts in §2 were measured against
compiler `3be06cc` with a throwaway spike, not reasoned from documentation, and they are what decides the
design — the rest follows from them.

## 1. The problem, from the user's side

A unit test in this framework should be pure test-framework code plus the *mocking* of whatever effects
the code under test uses, so those effects can be asserted on. Nothing else. In particular a unit test
should never name a carrier, a platform, or an effect implementation — it should be as ordinary to write
as a JUnit test, and it should work for whatever the code under test performs, because the platform it
will eventually run on accepts that too.

That is not what writing one costs today. To unit-test `eliot.build.Cache` — six functions over
`{Process, FileSystem, Throw[IoError], Throw[GitError]}` — a project must first write:

| What | Lines | Why |
|---|---|---|
| `data Fake[A]` + `implement Effect[Fake]` | ~20 | a carrier to run production code at |
| `implement Process[Fake]` | ~10 | 2 methods |
| `implement FileSystem[Fake]` | ~15 | **12 methods, of which the suite calls 3** |
| `implement Throw[GitError, Fake]`, `implement Throw[IoError, Fake]` | ~8 | one per failure channel |
| `data World` + builders + journal + rendering | ~90 | the world the mocks answer from |
| discharge words (`journalOf`, `answerOf`) | ~15 | to run a body and read it back |
| **total, before a single test is written** | **~195** | `eliot-build/test/eliot/build/FakeWorld.els` |

And it is written again, from scratch, in every project — `eliot-test`'s own `FakeConsole.els` is the same
195 lines with `Console` in place of `Process`. That is the actual barrier to writing unit tests here, and
it is a framework gap, not a language one.

## 2. What the language allows — measured

Four facts, each established by compiling it. They are stated first because three of the four are
*constraints*, and together they leave essentially one design.

1. **An ability may have at most one carrier-generic instance, full stop.** Instance identity is
   `(ability, pattern)` and a bare binder `F` is one pattern, so a second `implement[F[_] ~ Recorder]
   Fetcher[F]` beside `implement[F[_] ~ Suspend] Fetcher[F]` is rejected — *"Name was already defined in
   this module"* — even though no carrier could ever satisfy both constraints. **So the appealing symmetric
   design is impossible today**: an ability cannot carry a "real" instance for carriers that suspend and a
   "mock" instance for carriers that record. Mock instances must be at a **concrete** carrier.
2. **An instance must be colocated with its ability or with one of its type arguments; a third module is
   silently not found** (*"No ability implementation found for ability 'Fetcher' with type arguments
   [Mock]"*). So the mock instance for a base ability can only live with the base ability (the stdlib) or
   with the mock carrier — which means **whoever owns the mock carrier must write every base mock, and no
   test module can write one**. A per-project fixture is therefore not a shortcut a project chose; it is
   the only thing a project *can* do, and it is why the 195 lines recur.
3. **One generic instance covers every failure channel there will ever be.**
   `implement[E ~ Show] Throw[E, Mock]` type-checks and resolves, including for an error type declared in
   the *project under test* — verified with a project-side `data MissingPage` raised through a
   framework-side instance. This matters twice over: a project could not have written that instance itself
   (fact 2), and it means assertions ride the mock carrier for free, since `AssertionError` is just another
   `E`. Interleaved assert-during-a-mocked-run comes out of the same instance with no extra machinery.
4. **A mocked run is written inline**, because a slot declared with the capture tag `{| Mock} A` hosts the
   computation (W3, shipped 2026-09-04) — either as the argument of a helper inside a `pure` body, or as the
   whole body of a discharge word standing where `pure` stands. No definition per scenario, and the suite's
   row stays `{Writer[List[TestResult]]}`. Verified together with fact 3: a spike case asserts a value, then
   `calls`, then acts again, then `calls` again, all inside one mocked body.

Read together: **mocking cannot be done well by a project, so it has to be done once by the framework.**

## 3. The design

### 3.1 What the framework ships

```
eliot.test.World     the recording: a journal of calls, plus a typed script per base effect
eliot.test.Mock      data Mock[A](runMock: World => Pair[Either[Failure, A], World])
                     implement Effect[Mock]
                     implement[E ~ Show] Throw[E, Mock]          -- every failure channel, incl. assertions
                     implement Console[Mock] Process[Mock] FileSystem[Mock] Environment[Mock] Log[Mock]
eliot.test.Mocked    the discharge words a case uses: mocked / answerOf / callsOf
```

The split is forced: `Mock` names `eliot.file.File` (for `FileSystem` and `Path`) and `Mocked` names
`eliot.test.Assertion`, and **no file may import both** — they each export a `message`
(`eliot-build/docs/effectful-modules.md` §9.5). The generic `Throw` instance never names `AssertionError`
(`E` is a binder), so it stays on the `Mock` side.

**The split alone is not enough, which is why §5 opens with a rename.** The carrier's failure slot has a
type, and if that type is `AssertionError` then `Mock` must import `eliot.test.Assertion` and the collision
is back. Carrying the failure as *rendered text* instead compiles — the spike does exactly that — but the
runner then reports a failed mocked case as one line, `Failed(NotEqual(two, one))`, instead of the
expected/actual lines it renders for every other case. Losing the failure report in precisely the tests
this plan exists to make easy is not an acceptable trade, so the rename is stage 0 rather than a wish.

Everything here is platform-independent: the base effect abilities are abstract, so the mocks are ordinary
base-layer code and the framework still names no platform.

### 3.2 What a test writes

**`mocked(world, { … })` is the shape to lead with**: it *runs* the code under test on the mock carrier
against `world`, once, and the body then asserts on whatever it likes — the value that came back, and the
calls that were made, in the same run.

```eliot
import eliot.test.Assertion
import eliot.test.Test
import eliot.test.Mocked
import eliot.build.Cache

def testCases: {Writer[List[TestResult]]} Unit = {
   "publishedTags" should "clone a mirror it has not seen, then read what it published" in mocked(
      spawning("ls-remote", listing), {
         val published = publishedTags(cacheRoot, cached)

         published.size shouldBe 2
         calls shouldBe expectedFirstVisit
         publishedTags(cacheRoot, cached)
         calls shouldBe (expectedFirstVisit ++ "; " ++ expectedListing)
      })
}
```

That is act-then-assert, the way a JUnit test with a mock reads, and it is worth being the default for two
reasons beyond familiarity. Assertions are made **on the values themselves** — `published.size shouldBe 2`,
not `renderedTags(…) shouldBe "v1.2 aaa1, v1.3 bbb2"` — so the rendering helpers every fixture grows today
(`renderedTags`, `renderedText`) stop existing; the body is inside the carrier, so `shouldBe` works on any
`Eq & Show` as usual, and it works because of fact 3. And the run happens **once**, so a case that checks
both the answer and the interaction does not run the code twice.

`callsOf(world, computation)` and `answerOf(world, rendering, computation)` remain as conveniences for the
one-line cases — `callsOf` answers what a run *did*, `answerOf` what it *yielded or raised*, both by running
the computation on the mock exactly as `mocked` does. They are deterministic and pure, so a case that uses
both simply runs twice; harmless, but a reason not to lead with them.

For an expected failure, the mock side needs its own word — call it `raising(error, world, { … })`. The
framework's `expect` cannot serve here: it supplies a `Throw[E]` layer, which stacks a `ThrowCarrier` over
the mock carrier, and a mock instance is monomorphic and earns no lift through it (the standing
do-not-stack-over-a-fake rule).

### 3.3 The scripting vocabulary

One builder set on `World`, generalising `FakeWorld`'s: `spawning(fragment, output)` /
`spawnFailing(fragment, code, diagnostics)` for `Process`; `withDirectory(path)` / `withFile(path, content)`
for `FileSystem`; `reading(lines)` for `Console`'s input; `withVariable(name, value)` for `Environment`.
Everything else answers a documented default and is recorded, so an unscripted call is visible in `calls`
rather than silently plausible.

**Decision to take before building §3.3**: whether an unscripted call answers a default (lenient, what
`FakeWorld` does today) or fails the case (strict). Recommendation: **lenient plus a `strictly(world)`
switch** — the journal assertion already catches a surprise call, and lenience is what keeps a
twelve-method ability from forcing twelve lines of script per test.

## 4. What a project still writes, and why

**Its own abilities.** `implement PackageSource[Mock]` may live with `PackageSource` (production code
importing the test framework — rejected) or with `Mock` (the framework, which has never heard of it). So a
project mocking *its own* ability declares its own small carrier, exactly as `TablePackages` does today —
but with the framework's machinery to lean on it is the ability's instance and little else, not 200 lines.

**And it must, in one case, whatever we do**: a project ability whose production instance is a constrained
catch-all (`implement[F[_] ~ Process & FileSystem & …] PackageSource[F]`) *also matches* the framework's
wide `Mock`, and two candidates is "Multiple ability implementations found". That is rule 3 of
`effectful-modules.md` §3, and it is a good reason for the narrow carrier to stay a documented shape rather
than an embarrassment.

**The general fix is a language change**, and it is fact 1: if instance identity included the constraints —
or if two same-pattern instances were allowed when no carrier can satisfy both, which
*constraint-aware declination* already computes — then an ability could ship a `~ Suspend` instance and a
`~ Recorder` instance side by side, a project could do the same for its own abilities, and a mock carrier
would need no instances at all. That is the endpoint worth wanting. It needs a coherence story (what makes
two constraint sets provably disjoint?) and belongs in the compiler's own plan, not this one.

## 5. Stages

Each stage is green on its own and each deletes more than it adds.

- **Stage 0 — rename one of the two `message`s.** `eliot.file.File`'s `message(e: IoError)` or this
  framework's infix `message`; the second has the smaller blast radius (test code only, and `describedAs`
  reads better anyway). Without it the mock carrier cannot hold an `AssertionError`, and every failure in a
  mocked case degrades to one rendered line. One rename, then §3.1's split is a tidiness choice rather than
  a workaround.
- **Stage 1 — the carrier and one effect.** `World`, `Mock`, `Effect[Mock]`, the generic `Throw[E, Mock]`,
  `implement Console[Mock]`, and the three discharge words. Dogfood by rewriting this repository's own
  `test/eliot/test/example/` on it: `FakeConsole.els` (~100 lines) disappears, `GreeterTests` keeps every
  case. Success criterion: the four styles of `GreeterTests` still read the same, minus the fixture — and a
  failed assertion inside a `mocked` body reports expected/actual like any other.
- **Stage 2 — the rest of the base effects.** `Process`, `FileSystem`, `Environment`, `Log` mocks and their
  scripting. Migrate `eliot-build`: `FakeWorld.els` (195 lines) is deleted, `CacheTests`/`GitTests` keep
  their cases and their assertions, `TablePackages` stays (§4). Success criterion: 146 cases still green,
  and the diff is almost entirely deletion.
- **Stage 3 — interaction assertions.** `calls` as an ability on the mock carrier (mid-run), plus the small
  vocabulary a mock wants: "never spawned", "spawned exactly once", call ordering. Only now, because §2's
  fact 3 makes the plumbing free and stage 2's migration says which assertions are actually wanted.
- **Stage 4 — the user-ability shape.** Document the narrow-carrier recipe with the framework's machinery,
  and open the language question of fact 1 with the compiler.

## 6. Non-goals

- **Integration tests are not this.** A test that really spawns a process is not a unit test; it names the
  platform's run boundary deliberately (`eliot-build/test/eliot/build/RealWorldTests.els`) and it is the
  only kind of test that may. Nothing in this plan touches it.
- **No compiler change in stages 1–3.** Everything above compiles today; stage 4 only *asks* the question.
- **No widening of `eliot.test.Runner`'s row.** A mocked suite performs nothing, so `{Console}` stays
  correct — which is the point of doing it this way.
