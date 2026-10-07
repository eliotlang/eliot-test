# The Mock Doubles: a Review

> A review of `eliot.test.Mock` as of 2026-10-07: where the doubles answer differently from the platform they
> stand in for, what they cannot express, and what is odd in how they are built. `docs/mocking.md` is how the
> design came about; `README.md` ("Mocking") is how to use it. Each finding names the code it is about and
> ends in a task in §4, whose status is kept there.

The shape is sound and none of this argues for changing it: five named doubles, one `Recording` threaded
through `State`, every operation `recorded(update, answer)`, calls matched by words. The findings are about
**fidelity** — a double that answers where the platform raises makes a test green and production red — and
about a stringly verification surface that leaks one word into another.

## 1. Fidelity: where the double and the platform disagree

The platform is the jvm layer (`eliot/jvm/.../classgen/processor/FileNatives.scala`), read for each operation.

| | Operation | The double did | The platform does | Task |
|---|---|---|---|---|
| a | `delete` | removed the path **and everything under it**, and answered quietly for a path nobody put there | `Files.delete`: refuses a non-empty directory and a missing path | 1, done |
| b | `readLines`, `foldLines` | `split("\n", …)`, which keeps empty pieces: `"a\nb\n"` read as `a, b, ""`; `\r\n` left a `\r` on every line | `readAllLines` / `BufferedReader.readLine`: a final terminator ends the last line rather than starting an empty one, and `\n`, `\r\n` and `\r` all end a line | 1, done |
| c | `readFile` | answers `""` for a missing file and for a directory | raises `IoError` | 3 |
| d | `writeFile`, `appendFile` | write under a parent nobody created, which then exists | `writeString` raises `NoSuchFileException` | 3 |
| e | `writeFile` onto a directory, `createDirectories` onto a file | swap the entry's kind and keep the children, so the tree can hold a file with something inside it | raise | 3 |
| f | `foldCodePoints` | answered `initial` unchanged — a wrong answer that reads as a right one | folds the code points | 4, done |
| g | relative paths | the tree keys a path as shown, so `withWorkingDirectory("/work")` does not make `path("x")` mean `/work/x`; only a trailing `/` is normalised (`./a`, `a//b`, `..` are not) | resolves against the working directory | 10 |
| h | `whenSpawningCreates` | leaves the directory behind even when the matching `whenSpawning` says the command failed | a failed `git clone` leaves nothing | 9, done |

**The framework is not platform-independent.** `succeeding`, `failing` and `exiting` construct
`ProcessResult(…)`, a constructor only the jvm layer declares — eliot `v0.7`'s base has `type ProcessResult`
and its three accessors and nothing to build one with. So `root`, documented in `eliot.pkg` as having no
platform on purpose, compiles only in a closure that mounts `//jvm`. Task 2.

**Why the double refuses with a failed case.** The standard library has `ioError(message)` on eliot's
`master` (`b0fd81ba`, 2026-10-04), but no tag carries it yet, so the doubles still cannot raise `IoError`. Where
the platform would raise and the double cannot, the double **fails the case** with an `AssertionError` naming
the call and what the platform would have done. That is a stopgap with the right direction: a test that relied
on the lenient answer is told so, rather than passing. Task 3 turns those failures into `IoError`s once a tag
has the constructor, which also lets a test arrange a failure and assert that the code under test handles it.

## 2. Gaps: what a test cannot say

- **No arrangement can make a call fail.** No failing file operation, and no command that fails to *start* —
  which is the curl→wget fallback eliot-build's `ShellAssetsTests` documents as uncoverable. Waits on task 3.
- **Reading the tree back is itself recorded.** A test that checks what the code under test wrote
  (`readFile`, `exists`, `walk`) goes through the double, so `calls`, `onlyTheseWereCalled` and
  `nothingWasCalled` answer differently depending on whether they are asked before or after the check. Task 5.
- **A project's own double cannot journal a call.** `Mocking` and `Calls` take this module's private types, so
  eliot-build's `TableGit` journals through `Log`, and its operations appear as `log cloneMirror …`, interleaved
  with whatever the production code really logged. The same leak runs the other way: the public
  `recordedCalls` answers a `List[Call]` of a private type. Task 6.
- **One answer per command.** Code that runs one command twice in a single act — a retry, a fallback — cannot be
  answered "fail, then succeed"; re-arming only works between acts. Task 8.
- **A command could only leave directories behind.** `whenSpawningCreates` could not model `curl -o file` or
  `unzip` producing files, which is what eliot-build's asset hashing then reads. Task 9, done: `whenSpawningWrites`.
- **`writeFile` records the path, not the content,** so what was written is only checked by reading it back —
  which is recorded (above). Task 11.
- **`runInheritingIo` drops the arranged output**, which the platform sends to the terminal. Task 11.

## 3. Weirdnesses in how it is built

- **One namespace for operations and arguments.** Matching is over the call's description, so
  `wasCalled("delete")` matches `printLine delete this`, `whenSpawning("git fetch")` matches
  `git commit -m "git fetch"`, an argument `"a b"` is indistinguishable from two, and a directory with a space
  or an argument ending `$` confuses the `dir$ command` form. eliot-build relies on the blur:
  `wasCalledOnce("cloneMirror")` matches `log cloneMirror …`. Task 7.
- **A blank fragment matches everything** (`isBlank(fragment) || …`): `wasNeverCalled("")` always fails and
  `whenSpawning("", …)` is a catch-all. Neither is documented. Task 7.
- **Three re-arming rules.** `whenSpawning` is most-recent-wins, `whenSpawningCreates` applied *every* matching
  arrangement, and `whenReading` replaces the input queue. Task 9 made the second most-recent-wins too;
  `whenReading` still replaces, which for a queue is the same thing.
- **`forgetCalls` is an arranging word** (it performs `Mocking`) though it is about verification.
- **Everything verified is text.** `calls` joins with `"; "` because the base has no `Eq`/`Show` for `List`,
  and an argument holding `"; "` makes the text ambiguous. Task 12.
- **Small things:** the journal is appended to (`noted`), so a case is quadratic in its calls; `entriesOf`
  deduplicates quadratically; `countMismatch` says "exactly 1 calls".

## 4. Tasks

In rough priority order. "eliot" marks a task that needs an eliot change and a tag first.

1. **Done (2026-10-07).** `delete` refuses a non-empty directory and a path nothing is at; `readLines` and
   `foldLines` end lines the way the platform does. §1a, §1b.

   **What it costs a consumer.** Run against eliot-build's suite (a scratch tag, through `insteadOf`), it fails
   9 of 319 cases, all in `ShellAssetsTests` and all for one reason: `curl --output <archive>` is mocked, so the
   archive is never written, and `unpacked`'s `delete(archive)` now meets a path nothing is at. The production
   code is right — the real curl leaves the file — and the tests were green only because the old `delete`
   answered for anything. eliot-build pins eliot-test `v0.2` in its lock, so nothing breaks until it moves; when
   it does, either each of those cases arranges the archive (`withFile(path(expectedTree ++ ".download"), "")`)
   or task 9 lets the arrangement say what curl leaves behind. Task 9 is the better fix and should come first.
2. *eliot.* Add `processResult(exitCode, standardOutput, standardError)` to the base, body-less there and
   bodied in jvm like `ioError`, and build `succeeding`/`failing`/`exiting` with it.
3. *eliot (a tag carrying `ioError`).* Raise `IoError` where the platform does (§1c–e, and the refusals tasks
   1 and 4 made failed cases), add `whenFailing(fragment, message)` for file operations and a way to make a
   spawn fail to start, cover the wget fallback in eliot-build, and drop the "cannot raise `IoError`" notes from
   `Mock.els`, `README.md` and both repositories' `CLAUDE.md`.
4. **Done (2026-10-07).** `foldCodePoints` fails the case, naming the call, instead of answering `initial`. §1f.
5. Inspection that is not a call: `fileContentAt`, `fileExistsAt`, `filesUnder` over the tree, recording
   nothing, and the README telling a test to check with them.
6. A public `recordCall(operation, arguments)` on `Mocking`; move eliot-build's `TableGit` off `Log`; stop
   answering the private `Call` from a public operation.
7. Structured matching: the operation matched on its own, an argument holding spaces quoted in the
   description, and a blank fragment refused rather than matching everything.
8. Answers in turn: `whenSpawningInTurn(fragment, results)`, consumed like the console input.
9. **Done (2026-10-07).** `whenSpawningWrites(fragment, file, content)` leaves a file behind; what a command
   leaves (a directory or a file) is left only when it exits 0, and by the most recent matching arrangement of
   either word. §1h.

   **What eliot-build needs when it moves past eliot-test `v0.2`.** Checked on a scratch tag: one shared
   arrangement in `ShellAssetsTests`, `downloading` — `whenSpawningWrites("curl", path(expectedTree ++
   ".download"), "")` — called first in each mocked case, makes the suite 319 green again; nothing else changes.
10. Paths resolved against `withWorkingDirectory`'s directory and normalised (`.`, `..`, `//`) in `keyOf`.
11. The written content in `writeFile`/`appendFile`'s description (shortened), and `runInheritingIo`'s
    arranged output recorded as console output.
12. `calls` as a list once the base has `Eq`/`Show[List]`; until then, `;` escaped in the joined text.
13. `docs/mocking.md` §3.3 recommended "lenient, with a `strictly` switch later"; tasks 1, 3 and 4 move the
    doubles towards the platform's strictness instead. Record that as the decision.
