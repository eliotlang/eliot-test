# eliot-test

A unit-testing framework for the [Eliot language](https://github.com/robertbraeutigam/eliot), written in Eliot.

## Writing tests

A test module declares one value named `testCases`. The runner finds every such value in the program, so there is
nothing to register:

```eliot
import eliot.test.Test
import eliot.test.Assertion

def testCases: Test = {
   "greeting" should "address the name it was given" in {
      greeting("Bob") shouldBe "Hello, Bob!"
   }
   "greeting" should "not be the bare name" in {
      greeting("Bob") shouldNotBe "Bob"
   }
}
```

Each `"subject" should "expectation" in { … }` is one test case. A failing assertion stops that case only; the
rest still run.

### Assertions

| Word | Passes when |
|---|---|
| `actual shouldBe expected` | the two are equal |
| `actual shouldNotBe other` | the two differ |
| `condition.shouldBeTrue` / `.shouldBeFalse` | the `Bool` is `true` / `false` |
| `expect(error, body)` | `body` raises exactly `error` |
| `raising[E]("part of the message", body)` | `body` raises an `E` whose text contains the given part |
| `success` / `fail("reason")` | always / never |
| `assertion describedAs "message"` | the assertion passes; on failure the report shows the message and the original failure |

### What a test may do

The suite's return type says which effects its cases may perform:

```eliot
def testCases: Test = { … }               // the cases may assert, and nothing else
def testCases: {Console} Test = { … }     // the cases may also print, for real
```

### Mocking

`in mocked { … }` runs a case against test doubles of `Console`, `Log`, `FileSystem`, `Process` and
`Environment`. A case arranges what the doubles answer, runs the code under test, and checks the result and
what was called:

```eliot
import eliot.test.Mock

"greet" should "greet whoever the console offers" in mocked {
   whenReading(singleton("Bob"))                     // arrange

   greet                                             // act

   calls shouldBe "readLine; printLine Hello, Bob!"  // verify
}
```

- **Arranging:** `whenSpawning(fragment, succeeding(out) | failing(code, err) | exiting(code, out))`,
  `whenSpawningCreates`, `whenReading`, `withFile`, `withDirectory`, `withVariable`, `withArguments`,
  `withWorkingDirectory`. The most recent arrangement wins.
- **Verifying:** `wasCalled`, `wasNeverCalled`, `wasCalledOnce`, `wasCalledTimes`, `wasCalledAtLeast`,
  `wasCalledAtMost`, `wereCalledInOrder`, `nothingWasCalled`, `onlyTheseWereCalled`, plus `calls`,
  `callsMatching`, `callCount`, `lastCall` and `forgetCalls`.
- **Matching:** a fragment matches a call when its words appear in the call next to each other and in order.
  `wasCalled("git clone")` matches `/work$ git clone --mirror x`; `wasCalled("log")` does not match
  `printLine catalog`. A call reads as the operation and its arguments (`printLine hello`, `readFile /etc/hosts`),
  and a process as its directory, `$` and the command line (`/work$ git fetch`).
- **The file system** is an in-memory tree: what the test put there plus what the code under test wrote. A
  directory holding a file exists even if nobody created it.
- **Limitation:** the doubles cannot raise `IoError` yet, because the standard library offers no way to create
  one. A missing file reads as `""`.

For an effect of your own, write a named implementation in the test module and bind it with `with`; see the
`eliot-code` language guide.

## Running

```bash
./eliotw test          # compiles and runs this repository's own suites
NO_COLOR=1 ./eliotw test   # the same report without colours
```

The run prints one report per suite and a closing summary, and exits 1 if any case failed. A project uses the
framework by depending on its `suite` package next to its platform:

```
package test {
  at test
  dep github.com/eliotlang/eliot-test//suite v0.2
  dep github.com/robertbraeutigam/eliot//jvm v0.7
}
```

## Layout

| Module | What it is |
|---|---|
| `eliot.test.Test` | `TestCase`, `Outcome`, `TestResult`, the `Test` suite type, `should` and `in` |
| `eliot.test.Assertion` | `AssertionError` and the assertion words |
| `eliot.test.Mock` | the doubles, `mocked`, and the arranging and verifying words |
| `eliot.test.Report` | the report as plain data: tallies and the lines to print, in colour or plain |
| `eliot.test.Runner` | `main`: runs every suite and prints its report |

The framework tests itself: `test/src` holds its suites, written with the framework.
