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
