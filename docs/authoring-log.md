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

## 2026-09-30 · milestone 2 · part 2: accounts, snapshots, restart, init

Files: lib/std/login.wand, driver/{password,snapshot,init,engine,world,
main}.wand, wmd, and tests.

Observed
- Two more wand bugs, each filed:
  - #63: `core.Ctx(world = view w viewer, time = w.now)` failed with
    `core.World and world.World are not the same type`. A field read inside
    a qualified constructor's arguments resolved against that module's
    types. Reading `w.now` into a name first works around it.
  - #64: on restart the lever's `State.decoder` decoded into the player's
    `State`: `.name: expected String, got Int`. A silent wrong value when
    the fields line up. Fixed on the wand branch derived-by-module, not yet
    released; the snapshot tests need it.
- One test-helper mistake: a 7-character password in the sign-up helper
  was refused by the 8-character rule, and seven tests failed on it.
- A real run: sign-up with a date of birth, a 12-year-old refused and hung
  up on, a wrong password answered without saying which half was wrong, and
  after SIGTERM and a restart the lever's pulls went from 2 to 3.
- The admin's password prompt in `wmd init` shows what is typed: there is
  no IO.read_secret, and `stty -echo` would be another language.
- Tests: 63 pass, 5 runs of 5.

Easy, and why
- Derived `Account.encoder` and `File.decoder` made world.json one type
  and two functions.
- `Hash.hmac` and `Hash.equal?` made the password module 20 lines, with
  a constant-time compare.

Hard, and why
- #63 and #64 are the same kind of bug as #50: a name resolved in the
  wrong module's scope. Each file declaring its own `State`, as the design
  asks, is what finds them.

Candidate wand issues
- #64 needs a release before wmd can require it.
- IO.read_secret, from the design doc, is not filed yet.
- Later the same day: #63 and #64 were fixed and released in wand 0.95.2,
  and the #63 workaround (reading `w.now` into a name before building a
  `core.Ctx`) came out. Fixing #63 found one more place with #50's bug: a
  record update through a module, `core.World(base, n = v)`, built the
  other module's `World`. It had never been reachable, because such code
  did not typecheck before.

## 2026-09-30 · milestone 2 · part 3: details, props, hooks, socials, home

What happened
- The contract gained four Object fields (`details`, `props`, `on_enter`,
  `on_leave`) and four view functions (`detail_of`, `prop`, `awake?`,
  `home_of`). A field default must be a value, so the function fields are
  `Option`s and `details` is a map that defaults to `{}`.
- The rules on what mudlib code may ask for moved into one pure function,
  `engine.allowed`, which returns `Result String Unit`. Tests call it
  directly. New rules: no Move into a session, no Spawn of a body or of
  /std/login.
- Hooks run after any Move, and after a Spawn or a sign-up puts something
  into a room. A `hooks` flag on the Tx stops a hook's own moves from
  calling hooks again.
- Instances now keep `creator`, `created` and `modified`, and accounts keep
  `home` and `last_seen`. All are `Option`s with a default of None, so a
  world.json written before this change still reads. A test checks that.
- Driver commands: `/examine`, `/sethome`, `/who <name>`. Player verbs:
  `look at`, `home`, and ten socials in one table.
- Checker errors hit: 6
  - `constructor 'World' is missing fields 'detail_of', 'prop', 'awake?',
    'home_of'`. This was expected after the contract change. Fixed by
    adding the four functions to `engine.view`.
  - `type error: 224:17: expected World, got String`. `world.set_object`
    now takes the time, and the call in `apply` did not pass it yet. Fixed
    by passing `tx.at`.
  - `parse error: 277:1: unexpected token: )`. A splice of the new actions
    section left one `)` too many. Fixed by deleting it.
  - `namespace 'List' has no member 'flat_map' -- 'wand d List' lists its
    members`. Fixed with `List.flatten (List.map ...)`.
  - `type error: 526:21: the type allows {}, but the body performs Random`.
    A `change` body (`_examine`) called `_call`, whose guard handles
    Random. A change must be pure. Props are pure, so the call to them is
    now direct.
  - `type error: 16:10: 'ctx' needs its type before '.world' can be read:
    write '(ctx: Ctx)'`. The garden's `on_enter` lambda. Fixed with
    `(ctx: core.Ctx)`.
- wand s runs until green: 3.
- Test mistakes: 3. Each output line is its own outbox entry, and two tests
  expected one entry with "\n" in it. One test expected the lever's creator
  to be the driver, but the square spawns the lever, so the square is the
  creator. The comment on `creator` now says so.
- A real run over telnet: look at a detail, a social, the garden's hook,
  /sethome, home, /examine and /who. After SIGTERM, world.json has the home
  and the creator.
- Tests: 82 pass, 3 runs of 3.

Easy, and why
- Derived decoders filled in the new fields' defaults when the fields were
  not in the file, so the old snapshots need no migration.
- Putting the policy in one function made each rule one match arm, and a
  refusal a Result that a test can compare.

Hard, and why
- Nothing was hard in the language this session.

Candidate wand issues
- None new. `List.flat_map` is a common name that people will try first.

### Milestone 2 summary

Friction
1. Names resolved in the wrong module's scope: #50, #51, #63 and #64. Each
   mudlib file has its own `State`, as the design asks, and each time this
   found a new place in the checker or evaluator with the bug.
2. Stored functions and effects: #54 (a false V-BANG1) and #61 (a stored
   function ran effects that no manifest saw). The contract is records of
   closures, so this area was used on every object.
3. Map equality by insertion order (#62) made a test fail about once in
   eight runs, and it took a probe to see why.

Worked best
1. Derived encoders and decoders. Save, restore and world.json are one
   type each, and new `Option` fields with defaults read old files with no
   migration.
2. Effects as the gate on mudlib code. One handler guards Random, and a
   mudlib file cannot do IO because its manifest does not say so.
3. Pure world changes, applied in one `Shared.update`. The driver is one
   fiber, and the engine tests need no socket and no sleeps.

Proposed wand issues: `IO.read_secret` for the password prompt in `wmd
init`. Two quirks, not filed: two modules that claim no interface can share
a list, and `wand d --load` resolves imports from the working directory.
#50 to #64 are fixed and released in wand 0.95.0 to 0.95.2.

## 2026-09-30 · milestone 3 · the editor room, Check, Save and the gate

Files: lib/realm/editor/room.wand, lib/std/core.wand, driver/{files,engine,
world,scan,main}.wand, tools/blueprints.wand, driver/test_editor.wand.

Observed
- Checker errors hit: 11
  - `type error: these effects do not match: {} and {Raise, Random | ..}`,
    with no line or column. The editor was one `let make ... and ...` group
    of eight functions. Finding the call took about ten edits that each
    removed one part; the cause is a pure function called before its place
    in the group. Filed as wand #65, with a 9-line repro. Fixed by taking
    the commands out of the group: each returns a `Step` (the next state
    and the actions), and only `make` is recursive.
  - `type error: 206:17: '_run' is defined below, at line 211. Move the
    definition above its first use`. After the change above, the order of
    the functions matters. Fixed by moving `_run` up.
  - `namespace 'Map' has no member 'remove'`. It is `Map.delete`.
  - `parse error: 325:3: the ';' above ended the definition, so this line
    is a statement of its own rather than part of it`. An `and` body that
    sequences with `;` needs parentheses. Fixed with them.
  - `V-BANG1: 'listing' can raise`. Renamed `listing!`. The same for
    `_start`, renamed `_start!`.
  - `V-BANG2: 'check!' cannot raise`, and the same for `save!` and
    `draft!`. They return an Outcome for every failure. Dropped the `!`.
  - `V-BANG1: '_input_of' can raise`. It only returns the room's `input`
    hook, whose type has Raise; it calls nothing. Filed as wand #67. Fixed
    with `_takes_input?`, which returns a Bool, and a second lookup.
  - `'played' performs FS.Read, FS.Write, which the manifest does not
    allow`, in two test files. A step can now read and write mudlib files.
    Fixed by adding both to their `uses` lines.
  - `type error: 286:11: expected World, got Unit`. In a test, a `match`
    arm called `engine.restore`, which returns the old world (Shared.update
    returns the old value), and the other arm was `()`. Fixed with `let _ =
    Option.map ...`.
  - `did you forget to import the standard library FS?`, in a probe.
- wand s runs until green: 3. The first run of the editor tests had 2
  failures. One was my expectation: a check shows a warning for a file
  with no `uses` line. The other was wand #66: `Wand.check_at` could not
  import `../../std/core` from a new file in `lib/home/ann/`, because that
  directory did not exist yet. Workaround: the driver makes the directory
  before the check.
- A probe before writing the gate: `Checked.effects` is `["Random"]` for
  the lever and `["FS.Write"]` for a file that writes files. Raise is never
  in it. The gate is the check plus `effects ⊆ ["Random"]`, with no rewrite
  of the file's `uses` line.
- Design changes, each in a comment in the code:
  - The editor's `input` hook is always set. The design sets it only while
    lines are text. But ed commands such as `3,5p` are not words that the
    parser can match to a verb. The driver sends a line that starts with
    `/` to itself unless the room's `raw` prop is true, so /who and /quit
    work in the editor.
  - `Draft path source` is a third action, for `w!`.
  - `Remove id` is a new action: `q` takes the editor out of the world.
  - A save of a new blueprint writes driver/blueprints.wand again, or the
    next start would refuse to run. The code that writes the list moved
    from tools/blueprints.wand into driver/scan.wand, so the tool and the
    driver use one copy.
  - A move to a room that is gone sends a body home.
- Not done yet: falling back to the last editor that loaded cleanly needs
  hot reload, and so does the stdlib import allowlist.
- A real run on a copy of the repo: /edit a new file, `a` with a blank
  line and an indented line, `,n`, `t`, `w`, /who from inside the editor,
  and `q`. The file was saved formatted, and the list had the new
  blueprint. The server then restarted with it.
- Tests: 106 pass, 3 runs of 3. The 24 editor tests each use a copy of
  lib/ in a temporary directory.

Easy, and why
- `Wand.check_at` and `Checked.effects` made the check and the gate about
  30 lines. Error line numbers are buffer line numbers because the source
  is checked as typed and formatted only after it passes.
- The editor as plain steps (State in, Step out) was easy to test through
  the driver. Each ed command is one match arm.
- Derived `Outcome.encoder` and `Outcome.decoder`: the driver's answer to
  the editor is one type in core.wand.

Hard, and why
- #65. An error with no location in a 270-line file costs as much as any
  other error in this project. Four small repros written from a guess
  passed. The one that failed needed the direct `make` call in `_input`,
  found only by cutting down the real file.

Candidate wand issues
- #65, #66 and #67, filed today.
- Later the same day: #65, #66 and #67 were fixed and released in wand
  0.95.3. The cause of #65: a call to a group member whose body comes later
  was tied to the part of the caller's effects not yet known, and not to
  the part already known, so the member came out pure. The `mkdir` before
  a check and the Bool function for the input hook came out of wmd. The
  editor keeps its step design, because it is simpler to test.
