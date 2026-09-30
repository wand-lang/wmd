# Authoring log

What writing Wand was like for wmd: where it was hard, where it was easy, and
why. Observations (errors, counts, rewrites) stay apart from impressions, and
each impression points at evidence.

Each session adds one entry in this form:

```
## YYYY-MM-DD · milestone N · topic

Files: ...

Observed
- Checker errors hit: n (paste each message, and what fixed it)
- wand s runs until green: n
- Drift: habits from other languages that the checker caught
- Rewrites: what was written twice, and why
- Missing: what Wand did not have, and what was written instead
- Docs: where an answer was found, and where it was looked for first

Easy, and why

Hard, and why

Candidate wand issues
```

## 2026-09-29 · milestone 1 · plan, Wand module, Net test spike

Files: spikes/test_net_mock.wand, CLAUDE.md

Observed
- Checker errors hit: 2
  - `'++' joins two strings; two lists are joined with 'List.concat xs ys'`.
    I wrote `o ++ [text]`. Fixed with `List.concat o [text]`.
  - `parse error: 6:3: the ';' above ended the definition, so this line is a
    statement of its own rather than part of it -- put the body in
    parentheses to sequence it`. I wrote a top-level `let main () = a; b`
    over several lines in a scratch file. Fixed by mapping over a list.
- Lint warnings: 4 × V-BANG1 (a function that can raise has no `!`),
  1 × V-BANG2 (`reader!` cannot raise), 1 × A-USES1 (`Clock` declared but not
  yet used). All fixed by renaming or by the manifest.
- wand s runs until green: 1 failed to load (the `++` error), then green.
  Each later test addition was green on its first run.
- Drift: `++` to join two lists (1).
- Rewrites: none of the spike's logic. Names changed only for the linter.
- `wand f` rewrote `k (Ok ())` as `k Ok()`.
- Missing:
  - No typed way to make a fake `Connection`. A handler for `Net!listen`
    answers with any list, such as `[()]` or `["ann", "bo"]`, and each
    element becomes a "connection". `"%{conn}"` shows the fake value, so a
    handler can tell connections apart. This works because `Net!listen` has
    an open payload type.
  - `Wand.check` reads no files. A source that imports `../std/room` gives
    `E-IMPORT: Wand.check reads no files, so it cannot import one`.
  - `Wand.check` gives no effect set as data.
- Docs:
  - What a `Net!listen` handler must answer with is not in the reference.
    The reference lists the operation names only. Found it in
    `lib/evaluator.ml`: `"Net.listen: the handler must answer with a list of
    connections"`.
  - `wand = 0.93.0` in `wand.pkg` is the oldest wand accepted, not a pin.
    I first read it as a pin. Found in the reference, "Starting a package".
  - `wand run` does not exist: running a package script is `wand <url>`
    (`wand h`). The design doc said `wand run`; the doc is now fixed.
  - `Map` keys are always `String` (`wand d Map`). The design doc's
    `Map SessionId (List String)` needs `SessionId` to be a string.

Easy, and why
- The effect gate needs no new wand code. `Wand.check` on source with
  `uses {Random, Raise}` as its manifest refuses `IO` at top level and
  `FS.Write` hidden in a closure in a list, and the error names the effect:
  `'g' performs FS.Write, which the manifest does not allow`.
- Handlers over `Par` worked on the first try: both fibers of a session, and
  eight sessions under `Par.each_stream`, reached the one handler. The `Par`
  module says so ("A worker is never outside a handler's reach").
- `Shared` inside handler cases gave scripted input and collected output
  with no extra machinery.

Hard, and why
- Getting a `Connection` into a test. The first idea (mock `Net!read_line`
  and `Net!write` only) needs a connection value to pass in, and
  `Connection` has no constructor. The answer needed a read of the evaluator
  (see Docs).
- The design has a bug that the spike found: when the reader ends,
  `Par.all!` stops the writer, and lines still in the outbox are lost. The
  test "without a last drain, Par.all! stops the writer before it writes"
  shows it. A player who types `quit` would not see the goodbye line. The
  fix is one more drain after `Par.all!` returns.

Candidate wand issues
- Document in the reference what `Net!listen` and `Net!accept` carry and
  answer, and show the fake-connection test pattern. Maybe add a typed fake
  connection for tests.
- `Wand.check_at path src`: check source as if it were the file at `path`,
  so its relative imports resolve.
- `Checked.effects`: the inferred effect set as data.

## 2026-09-30 · milestone 1 · Talk

Files: wmd, driver/telnet.wand, driver/world.wand, driver/session.wand,
driver/log.wand, driver/main.wand, and a test file for each of the first four.

Observed
- Checker errors hit: 7
  - `did you forget to import the standard library Map?` in session.wand.
    Fixed with `import Map`.
  - `this value needs its type before '.outboxes' can be read: bind it to a
    name that carries its type, 'let (x: World) = ...'`. I wrote
    `(Shared.get state).outboxes`. Fixed with a function `world.outbox` that
    takes a typed `World`. Later, in log.wand, the suggested
    `let (w: world.World) = Shared.get state` worked at once.
  - `performs Proc, which the manifest does not allow` in wmd, with the
    exact line to write. Fixed by pasting it.
  - `'flushed!' performs FS.Write, which the manifest does not allow` in
    test_log.wand. The test handles `FS!append` only, so `FS.Write` stays,
    as the reference says ("What a handler discharges"). Declared it.
  - `parse error: expected ->, got |` for an or-pattern
    (`| "look" | "l" -> ...`), in a probe. Wrote two arms.
  - `lex error: unknown escape \x` for `"\xff"`, in a probe.
  - `invalid regex` for `[\xfb-\xfe]`. `\xff` works in a regex outside a
    character class but not inside one. Fixed with an alternation,
    `(\xfb|\xfc|\xfd|\xfe)`.
- Lint warnings: 3 × V-BANG1, 1 × V-PRED3 (`_drain` returns `Bool`; now
  `_drained?`), 2 × V-USES2 (no manifest in log.wand and session.wand). All
  fixed as the message said.
- Test failures: 2 of 23 in test_world.wand on the first run. Both were my
  expectations, not the code: one test moved both players, and one counted
  a line that is not there.
- First runs: world.wand (about 250 lines) typechecked with no error. The 8
  session tests and the 3 log tests passed on their first run.
- `wand t` and `wand s` take a directory; `wand f driver/` gives
  `Error loading 'driver/': Is a directory`.
- A real run with two `nc` clients worked: an idle client saw the other
  speak and leave; IAC bytes were dropped; a client that dropped without
  `/quit` was removed; every event reached the log.
- On SIGTERM the server prints
  `Error: eval error: <stdlib>/Shared.wand:24:3: race: no thunk finished`.
  It prints the same with or without the final-flush bracket, so it comes
  from wand. The bracket's release still ran: all 4 events were logged.
- Missing:
  - `Shared.update` gives back `Unit`, so a fiber cannot get a value out of
    an atomic change. Three workarounds follow from it:
    - A session id is `session:` plus `Random.hex 12`, not a counter,
      because a fiber cannot learn which number an update gave it.
    - Events queue in the world, and one logger fiber takes them out.
    - A writer takes its outbox with a get, then an update that drops what
      it took. This is safe only because it is the one fiber that removes
      lines.
  - Strings have no byte escapes. Test input with IAC bytes is written in
    Base64.
  - No or-patterns in `match`.
- Docs: `wand d` answered every signature question (List, Map, String,
  Option, Path, FS.mkdir, Random.hex, Resource.make) without opening the
  reference.

Easy, and why
- A pure world with the world as the last argument made every command a
  pipeline, such as `w |> tell ... |> _put ... |> look s.id`, and every test
  a pipeline too.
- Handler tests covered what a unit test usually cannot: the goodbye that
  arrives after `/quit`, a connection whose writes fail, two sessions at
  once. None needed a socket (see the 2026-09-29 entry).
- Derived codecs. `Opts.decoder` made the flags, with the error
  `.port: expected Port, got "bad"`. `Event.encoder` made each log line and
  left out every field that is `None`.
- The manifest errors give the exact line to write.

Hard, and why
- Deciding where a value comes out of a `Shared.update` (see Missing). This
  shaped the design more than any other single fact.
- Bytes: see the three checker errors about `\x`.

Candidate wand issues
- `Shared.modify : Shared 'a -> ('a -> ('a, 'b)) -> 'b`: an update that
  gives a value back.
- Byte escapes in strings, and `\xNN` inside a regex character class.
- A clear message on SIGTERM, not `race: no thunk finished` at
  `Shared.wand:24`. Fixed on 2026-09-30; see the entry below.
- `wand f` should take a directory, as `wand t` and `wand s` do.

### Milestone 1 summary

Friction
1. `Shared.update` gives back nothing, which forced random session ids, an
   event queue, and a single-remover rule for each outbox.
2. Bytes in source: no string escapes, and `\x` not allowed in regex
   character classes.
3. A field needs a known type, and a handler removes an effect only when it
   handles every operation of it. Both are right, and both cost a round
   trip.

Worked best
1. Effect handlers over `Par`: whole sessions tested with no socket.
2. Pure code with the world last, plus record update syntax.
3. Messages that say the fix: manifest lines, `!` and `?` names, the
   `List.concat` hint for `++` on lists.

Proposed wand issues: `Shared.modify`; byte escapes; the SIGTERM message;
`wand f` on a directory; the `Net!listen` docs; `Wand.check_at`;
`Checked.effects`. None is filed yet.

## 2026-09-30 · milestone 1 · the SIGTERM message, fixed in wand

Files: ../wand lib/evaluator.ml, test/test_signals.ml, CHANGELOG.md
(branch `par-all-stops` in wand)

Observed
- Cause: `Par.all!` is a race. On SIGTERM every branch raised
  `Interrupted 143`. The race counted that as neither a result nor a
  failure, so no branch won, and `Par.all!` raised `race: no thunk
  finished` as an ordinary error: exit 1, not 143.
- A second bug with the same cause: `Proc.exit 3` in one branch was lost.
  The race waited for the other branch (2 s in the reproduction), then the
  script went on and exited 0.
- Fix: the race tells a stop (a signal code, or an `Interrupted` in an item
  whose cancel flag is not set) from a lost race (flag set, code 0). A stop
  cancels the other items and is raised again after they are joined.
- Two new tests in test_signals.ml. Both failed before the fix and pass
  after it. The full wand suite, 1,951 wand-level tests, the format and
  docs checks, and every demo pass.
- wmd with the fixed wand: SIGTERM exits 143 with no message, and the last
  events are logged.

## 2026-09-30 · milestone 2 · objects, the engine, and wand 0.95

Files: lib/std/{mud,room,text,player,login}.wand,
lib/realm/commons/{square,garden,lever}.wand, driver/{world,engine,parse,
scan,session,log,main}.wand, tools/blueprints.wand, and tests.

Observed
- wand bugs found while writing this, each filed and fixed in wand 0.95:
  - #50: a record update found its type by bare name in one table for the
    whole program. The lever's second pull built the player's `State`:
    `constructor 'State' has no field named 'pulls'`. With matching fields
    it would have been a wrong value with no error.
  - #51: `implement mud.Blueprint` failed with `unknown type 'Object'`, no
    location, because the interface's member types were read in the
    implementing file. Worked around with a type parameter until the fix.
  - #52: a list of two different blueprint modules was refused with
    `expected a module implementing mud.Blueprint Init, got a module
    implementing mud.Blueprint Init`.
  - #53: a state-passing Random handler stopped wand with
    `Fatal error: exception Stdlib.Effect.Continuation_already_resumed`.
    That ended the plan to run mudlib code inside Shared.update; the driver
    runs it in one fiber instead, as the design says.
  - #54: a false V-BANG1 on the lever's `make`.
  - #61: a function stored in a record field could run a command that no
    manifest saw. A driver under `uses {IO}` ran `$(echo pwned)` built in a
    file under `uses {Random, Raise}`. Found while fixing #54.
- The proposals from milestone 1 were built too: `Shared.update` answers
  the old value (#55), `\xNN` escapes (#56), `wand f <dir>` (#57),
  `Wand.check_at` (#59), `Checked.effects` (#60).
- After 0.95, every workaround came out: plain `implement mud.Blueprint`;
  one plain list of blueprints; one-step takes of an outbox and the event
  queue; counted session ids; `\xff` in tests and `[\xfb-\xfe]` in the
  telnet regex.
- Other checker errors hit: `'Command' is a built-in type, so it cannot be
  declared` (renamed `Parsed`); `mud.World and world.World are not the same
  type` inside a record construction, gone when the annotated lambda moved
  to its own function; a record field `List (World -> World)` gathered
  Random from its uses until written `! {}`.
- One regression the tests caught: with readers that only queue, a line
  sent after `/quit` still ran. The engine now ignores a closing session.
- Tests: 45 pass. The engine tests ran green on their first run.

Easy, and why
- Mudlib objects as records of closures over a private State record: each
  file was short, and derived `State.encoder` and `State.decoder` made save
  and restore one line each.
- Effects as the gate: the Random handler, and the test that a mudlib seed
  cannot fix the driver's dice, took ten lines.

Hard, and why
- Interfaces across modules (#51, #52): the first real use of interfaces
  with several implementing files found both bugs in an hour.
- Where to run mudlib code: pure inside Shared.update needed a stateful
  handler (#53); outside it needed a way to learn what an update changed
  (#55). One driver fiber avoids both.

Candidate wand issues
- None open. Two older quirks noted, not filed: two modules that claim no
  interface can share a list; `wand d --load` resolves a file's imports
  from the working directory, not the file's.

## 2026-09-30 · milestone 2 · a flaky test, and wand 0.95.1

Observed
- The spike test "two connections" failed about once in eight runs:
  `expected {ann = [...], bo = [...]}, got {bo = [...], ann = [...]}`. The
  two maps held the same entries; the sessions had finished in the other
  order. A probe showed `{a = 1, b = 2} == {b = 2, a = 1}` was `false`, and
  `List.unique` kept both. Filed as wand #62 and fixed in 0.95.1: maps are
  equal by keys and values, and still print in insertion order.
- wand.pkg now needs 0.95.1. The suite passed 10 runs of 10 with it.
- `mud` was renamed `core` (lib/std/core.wand). Entries above quote the
  old name as the errors said it.

Candidate wand issues
- None open.
