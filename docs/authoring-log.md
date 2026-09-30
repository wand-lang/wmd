# Authoring log

What writing Wand was like for wmd: where it was hard, where it was easy, and
why. Observations (errors, counts, rewrites) stay apart from impressions, and
each impression points at evidence. Bugs found and fixed, in wand or in wmd,
are left out: they say nothing about writing in the language.

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
    (`wand h`).
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
- wand s runs until green: 2. Two of 23 world tests failed on the first
  run, both on my expectations.
- First runs: world.wand (about 250 lines) typechecked with no error. The 8
  session tests and the 3 log tests passed on their first run.
- `wand t` and `wand s` take a directory; `wand f driver/` gives
  `Error loading 'driver/': Is a directory`.
- A real run with two `nc` clients worked: an idle client saw the other
  speak and leave; IAC bytes were dropped; a client that dropped without
  `/quit` was removed; every event reached the log.
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

Proposed wand issues: `Shared.modify`; byte escapes; `wand f` on a
directory; the `Net!listen` docs; `Wand.check_at`; `Checked.effects`.

## 2026-09-30 · milestone 2 · objects, the engine

Files: lib/std/{core,room,text,player,login}.wand,
lib/realm/commons/{square,garden,lever}.wand, driver/{world,engine,parse,
scan,session,log,main}.wand, tools/blueprints.wand, and tests. (core.wand
was first called mud.wand.)

Observed
- Checker errors hit: 2
  - `'Command' is a built-in type, so it cannot be declared`. Renamed the
    type `Parsed`.
  - A record field `List (World -> World)` gathered Random from its uses
    until the field type said `! {}`.
- wand s runs until green: 2.
- Tests: 45 pass. The engine tests ran green on their first run.

Easy, and why
- Mudlib objects as records of closures over a private State record: each
  file was short, and derived `State.encoder` and `State.decoder` made save
  and restore one line each.
- Effects as the gate: the Random handler, and the test that a mudlib seed
  cannot fix the driver's dice, took ten lines.

Hard, and why
- Where to run mudlib code. Pure code inside `Shared.update` needed a
  handler that keeps state; code outside it needed a way to learn what an
  update changed. One driver fiber avoids both.

Candidate wand issues
- None.

## 2026-09-30 · milestone 2 · part 2: accounts, snapshots, restart, init

Files: lib/std/login.wand, driver/{password,snapshot,init,engine,world,
main}.wand, wmd, and tests.

Observed
- wand s runs until green: 2. Seven tests failed on one mistake in a test
  helper: a 7-character password, which the 8-character rule refused.
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
- Nothing in the language.

Candidate wand issues
- IO.read_secret, from the design doc.

## 2026-09-30 · milestone 2 · part 3: details, props, hooks, socials, home

What happened
- The contract gained four Object fields (`details`, `props`, `on_enter`,
  `on_leave`) and four view functions (`detail_of`, `prop`, `awake?`,
  `home_of`). A field default must be a value, so the function fields are
  `Option`s and `details` is a map that defaults to `{}`.
- The rules on what mudlib code may ask for moved into one pure function,
  `engine.allowed`, which returns `Result String Unit`. Tests call it
  directly.
- Instances now keep `creator`, `created` and `modified`, and accounts keep
  `home` and `last_seen`. All are `Option`s with a default of None, so a
  world.json written before this change still reads. A test checks that.
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
- wand s runs until green: 3. The failures were in my test expectations:
  each output line is its own outbox entry, and the square, not the
  driver, makes the lever.
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
1. A value's type must be known before a field is read: `'ctx' needs its
   type before '.world' can be read`. A lambda over a record needs an
   annotation, and so did `(Shared.get state).outboxes` in milestone 1.
2. Function-typed record fields: a field's effects must be written out
   (`! {}`), and a field default must be a value, so each optional function
   is an `Option`.
3. Names: `Command` is taken by a built-in type, and `List.flat_map` does
   not exist.

Worked best
1. Derived encoders and decoders. Save, restore and world.json are one
   type each, and new `Option` fields with defaults read old files with no
   migration.
2. Effects as the gate on mudlib code. One handler guards Random, and a
   mudlib file cannot do IO because its manifest does not say so.
3. Pure world changes, applied in one `Shared.update`. The driver is one
   fiber, and the engine tests need no socket and no sleeps.

Proposed wand issues: `IO.read_secret` for the password prompt in `wmd
init`.

## 2026-09-30 · milestone 3 · the editor room, Check, Save and the gate

Files: lib/realm/editor/room.wand, lib/std/core.wand, driver/{files,engine,
world,scan,main}.wand, tools/blueprints.wand, driver/test_editor.wand.

Observed
- Checker errors hit: 9
  - `type error: 206:17: '_run' is defined below, at line 211. Move the
    definition above its first use`. The editor's commands are plain
    functions, so their order matters. Fixed by moving `_run` up.
  - `namespace 'Map' has no member 'remove'`. It is `Map.delete`.
  - `parse error: 325:3: the ';' above ended the definition, so this line
    is a statement of its own rather than part of it`. An `and` body that
    sequences with `;` needs parentheses. Fixed with them.
  - `V-BANG1: 'listing' can raise`. Renamed `listing!`. The same for
    `_start`, renamed `_start!`.
  - `V-BANG2: 'check!' cannot raise`, and the same for `save!` and
    `draft!`. They return an Outcome for every failure. Dropped the `!`.
  - `'played' performs FS.Read, FS.Write, which the manifest does not
    allow`, in two test files. A step can now read and write mudlib files.
    Fixed by adding both to their `uses` lines.
  - `type error: 286:11: expected World, got Unit`. In a test, a `match`
    arm called `engine.restore`, which returns the old world (Shared.update
    returns the old value), and the other arm was `()`. Fixed with `let _ =
    Option.map ...`.
  - `did you forget to import the standard library FS?`, in a probe.
- wand s runs until green: 3. One failure was my expectation: a check shows
  a warning for a file with no `uses` line.
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
  - A move to a room that is gone sends a body home.
- A real run on a copy of the repo: /edit a new file, `a` with a blank
  line and an indented line, `,n`, `t`, `w`, /who from inside the editor,
  and `q`. The file was saved formatted.
- Tests: 106 pass, 3 runs of 3.

Easy, and why
- `Wand.check_at` and `Checked.effects` made the check and the gate about
  30 lines. Error line numbers are buffer line numbers because the source
  is checked as typed and formatted only after it passes.
- The editor as plain steps (State in, Step out) was easy to test through
  the driver. Each ed command is one match arm.
- Derived `Outcome.encoder` and `Outcome.decoder`: the driver's answer to
  the editor is one type in core.wand.

Hard, and why
- Order. A function must be defined above its first use, so the editor's
  functions were reordered, and an `and` group is the only way to call
  forward.

Candidate wand issues
- None new.

## 2026-09-30 · milestone 3 · events, the current line, and the end of the milestone

Files: driver/engine.wand, lib/realm/editor/room.wand, driver/test_editor.wand,
driver/test_session.wand.

Observed
- Checker errors hit: 0.
- wand s runs until green: 3. The first run had 2 failures, both mine:
  the spawn of the lever at boot is now an event, and two tests listed
  events without it.
- After `w`, `.` names the same line of code: the editor counts the lines
  with text, because blank lines are what formatting adds and takes away
  most.
- The generated import list is the weak point, as the user pointed out.
  It exists because an import is fixed when a file is checked, so new code
  loads only after a restart, and the driver writes into its own source
  directory. `Wand.load` in milestone 4 removes it.
- Tests: 109 pass, 3 runs of 3.

### Milestone 3 summary

Friction
1. The generated import list: new code needs a restart, and the driver
   writes its own source. There is no way in Wand to load a file at run
   time yet.
2. Definition order: a function must be defined above its first use.
3. `!` naming churn: the linter asked for `listing!` and `_start!`, and
   against `check!`, `save!` and `draft!`, as each function's risk changed.

Worked best
1. `Wand.check_at` and `Checked.effects`: the check and the effect gate
   are about 30 lines of driver code, and error lines are buffer lines.
2. The editor as plain steps, State in and Step out: its tests go through
   the driver.
3. Derived encoders and decoders again: the driver's answer to the editor
   is one type, and the editor's buffer survives a restart with no code
   for it.

Proposed wand issues: `Wand.load` with the effect gate, as the design
plans for milestone 4.
- Later: the test files moved into `test/` beside the code they test. Each
  import went up one more level (`./engine` to `../engine`), and `wand s`
  found the files in their new places with no change. 109 tests pass.

## 2026-09-30 · milestone 4 · Wand.load, /update and /reset

Files: driver/{mudlib,engine,world,main,init,files,scan}.wand, wmd,
lib/realm/editor/room.wand, driver/test/*. driver/blueprints.wand and
tools/blueprints.wand are gone.

Observed
- The API was chosen with the user: every interface has a derived
  `loader`, as a type has `decoder`, and `Wand.load! core.Blueprint.loader
  path` gives the file's module. A loaded file is held to the effects its
  interface's members allow. Before any code, a probe booted the wmd
  engine on blueprints loaded this way: sign-up, the lever and the
  garden's hook all worked.
- Checker errors hit in wmd: 1
  - `unbound variable 'raise' -- errors are values: return an Error, call
    a !-suffixed function to raise, or wrap a call with try`. Fixed with
    `Result.get! (Error ...)`.
- wand s runs until green: 5. The failures were in my tests:
  - 24 editor tests failed at once: they worked on a copy of lib/, and a
    file loads only when it imports the driver's own std/core.wand. The
    tests now use the real lib/, save under lib/home/ann/, and remove it
    after.
  - A test that changed `pulls` to a String: the new file did not check,
    so /update refused it before any state was read.
- A design decision with the user: the driver and the mudlib must share
  one core.wand, so a game is a checkout of wmd for now. The `wand <url>`
  layout stays possible if a game's mudlib imports core from the package
  (see the design doc, "One core.wand").
- A real run on a copy of the repo: `/edit lever` as admin, `,s/pulled/
  yanked/`, `w`: "Updated /realm/commons/lever: 1 instances on the new
  code.", and `look at lever` said "It has been yanked 1 times." Then
  `/reset lever` set it to 0.
- Tests: 118 pass, 3 runs of 3.

Easy, and why
- Taking `lib` out of the engine was one regular expression: it was
  passed in the same position everywhere. It is a field of the world now,
  so /update changes it in the middle of a step.
- The derived-member pattern (`State.decoder`) made `core.Blueprint.loader`
  a small change in the checker and needed no new syntax.

Hard, and why
- Identity by path. A loaded module shares core's types with the driver
  because it imports the same file, and for the same reason a copy of lib/
  could not be loaded at all.

Candidate wand issues
- FS has no way to make a symlink. A test with a symlinked std/ would
  not have needed the real lib/.

## 2026-09-30 · milestone 5 · part 1: fuel, caps, and objects that stop working

Files: driver/{engine,world}.wand, driver/test/test_editor.wand.

Observed
- Every mudlib call runs under `Wand.limit`: 100,000 steps and 1,000
  nested calls, including `short`, `long`, details and props, which the
  driver had called with no guard at all.
- Caps: a line longer than 8,192 characters is refused, a reply with more
  than 100 actions is refused whole, and a saved state over 64 KB (1 MB
  for /std/login) is left out of the snapshot. An object that faults 5
  times in a row stops working until /update or /reset.
- A 20,000,000-step loop took 4.57 s under a wand with the limit code and
  4.82 s under one without: the limit has no measurable cost.
- Checker errors hit: 0.
- The limit tests edit the real lever through the editor as admin: a
  `pull` that calls a function that calls itself forever was stopped
  with "ran out of steps after 100000", and `look` still answered.
- Tests: 123 pass, 3 runs of 3.

Easy, and why
- `let spin (n: Int) = spin (n + 1) and make (s: State) =`: the editor's
  `s` command could add a function to the lever's file, because an `and`
  group needs no new line.
- `Wand.limit` wraps a call and gives a Result, so the driver's guard for
  every mudlib call stayed one function.

Hard, and why
- Nothing in the language.

Candidate wand issues
- None new.
- Later: spikes/ was deleted. Its one file, the milestone 1 Net test,
  was covered by driver/test/test_session.wand, which tests the same way
  against the real session code. The design doc now points at that test.
