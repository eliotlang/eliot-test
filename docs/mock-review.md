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
- **Reading the tree back was itself recorded.** A test that checks what the code under test wrote
  (`readFile`, `exists`, `walk`) goes through the double, so `calls`, `onlyTheseWereCalled` and
  `nothingWasCalled` answered differently depending on whether they were asked before or after the check.
  Task 5, done: `fileContentAt`, `existsAt` and `filesUnder` read the tree and record nothing.
- **A project's own double could not journal a call.** `Mocking` and `Calls` take this module's private types, so
  eliot-build's `TableGit` journals through `Log`, and its operations appear as `log cloneMirror …`, interleaved
  with whatever the production code really logged. The same leak ran the other way: the public
  `recordedCalls` answered a `List[Call]` of a private type. Task 6, done: `recordCall`, and `Calls` answers
  descriptions.
- **One answer per command.** Code that runs one command twice in a single act — a retry, a fallback — could not be
  answered "fail, then succeed"; re-arming only works between acts. Task 8, done: `whenSpawningInTurn`.
- **A command could only leave directories behind.** `whenSpawningCreates` could not model `curl -o file` or
  `unzip` producing files, which is what eliot-build's asset hashing then reads. Task 9, done: `whenSpawningWrites`.
- **`writeFile` records the path, not the content,** so what was written is only checked by reading it back —
  which is recorded (above). Task 11.
- **`runInheritingIo` drops the arranged output**, which the platform sends to the terminal. Task 11.

## 3. Weirdnesses in how it is built

- **One namespace for operations and arguments.** Matching was over the call's description, so
  `wasCalled("delete")` matched `printLine delete this`, and a directory with a space or an argument ending `$`
  confused the `dir$ command` form. eliot-build relied on the blur: `wasCalledOnce("cloneMirror")` matched
  `log cloneMirror …`. Task 7, done: the first word is matched against what was called. What remains: the words
  after it float over the arguments, so `whenSpawning("git fetch")` still matches `git commit -m "git fetch"` and
  `"a b"` still matches as two arguments, and a fragment with no directory matches an operation and a program of the
  same name alike.
- **A blank fragment matched everything** (`isBlank(fragment) || …`): `wasNeverCalled("")` always failed and
  `whenSpawning("", …)` was a catch-all. Task 7, done: it matches nothing and is refused.
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
5. **Done (2026-10-07).** Inspection that is not a call: `fileContentAt` (`Option[String]`, `None` where there
   is no file), `existsAt` (a file or a directory, as `exists` answers) and `filesUnder` (as `walk` answers),
   read through a second operation of `Calls`, `recordedFiles`, and recording nothing. The README tells a test
   to check with them. `recordedFiles` answers the private `Entry`, the same leak `recordedCalls` has (task 6).
6. **Done (2026-10-07).** `recordCall(operation, arguments)` journals a call that reads like the framework's own,
   through a new `Mocking` operation. `Calls` answers no private type: `recordedCalls` is each call's description
   (what matching always compared), `recordedFiles` each path with its content, or none for a directory.
   `Mocking`'s operations still *take* private types — a test cannot construct one, so nothing reaches them but
   the words, and closing that would take one operation per arrangement kind.

   **eliot-build's side, checked on a scratch tag:** `TableGit` calls `recordCall` instead of `log`, its clause
   rows trade `Log` for `Mocking`, and the expectations in `CacheTests` and `GitPackagesTests` lose their `log `
   prefix (`"log tags …"` is now `"tags …"`). With task 9's `downloading` arrangement in `ShellAssetsTests`, the
   suite is 319 green. It lands when eliot-build moves past eliot-test `v0.2`, together with task 9's change;
   neither compiles against `v0.2`.
7. **Done (2026-10-07).** Structured matching. `Call` is one public record, `Call(callee, callArguments,
   callDirectory)` — an operation in no directory, a process as its program, the rest of its command line and its
   directory — and `Calls.recordedCalls` answers it, which also closes task 6's last leak. A fragment's first word
   must equal the callee, an optional `<dir>$` before it must equal the directory, and the rest float over the
   arguments' words. A blank fragment matches nothing: every verification refuses one, and the process double fails
   the case at a spawn while one is arranged (an arranging word cannot fail, being `{Mocking}` alone). The
   argument description is not quoted — `calls` reads as before — so `"a b"` and `a`, `b` still read and match
   alike; `recordedCalls` tells them apart. `countMismatch` now says "1 call".

   **What eliot-build changes when it moves past this tag:** every `whenSpawning…` fragment names the program
   (`"ls-remote"` becomes `"git ls-remote"`, `"show v1.9:eliot.pkg"` becomes `"git show v1.9:eliot.pkg"`), and a
   verification of a spawn does too (`wasNeverCalled("fetch")` asks about `TableGit`'s `fetch` operation, never a
   `git fetch`). `recordCall` operations (`cloneMirror`, `addWorktree`) and `printLine`/`delete` fragments are
   unchanged.
8. **Done (2026-10-07).** `whenSpawningInTurn(fragment, results)`: the most recent matching arrangement answers
   its next result and keeps the last; `whenSpawning` is it with one result. What a run leaves behind follows the
   answer that run got.
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
