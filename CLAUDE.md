# wmd

WMD (World of Many Doors, aka Wand MUD) is a MudOS-style MUD written in Wand.
The design doc is https://claude.ai/artifact/7LBnJra7BUMF4eHaaDFEqv. Read it
with the docs tools before you change the design.

## Rules

- Everything is Wand. The driver, the mudlib, the tests and every tool are
  `.wand` files. Do not write Python, shell scripts or a Makefile.
- When Wand cannot do something, do not use another language. Write it in
  Wand if you can, and record the gap in the authoring log. If the gap is
  general, propose a wand change.
- `wand t` typechecks. `wand s` runs the `test_*.wand` files. `wand f`
  formats. Run all three before you commit.
- A test file goes in a `test/` directory beside the code it tests, as
  `driver/test/test_engine.wand` for `driver/engine.wand`.
- `wand = ...` in `wand.pkg` is the oldest wand that wmd accepts. Change it
  only when wmd starts to use a newer feature.

## Authoring log

`docs/authoring-log.md` records what writing Wand was like. It is the main
output of this project. Add to it during every work session and before every
commit.

- Write one entry per session, in the form at the top of the log.
- Keep what you observed (errors, counts, rewrites) apart from your
  impressions. Each impression must point at evidence. A model is not a
  reliable narrator about its own processing.
- Paste each checker error that you hit, and write what fixed it.
- Leave out bugs that were found and fixed, in wand or in wmd. They say
  nothing about how good the language is to write in.
- At the end of each milestone, add a summary: the three biggest sources of
  friction, the three things that worked best, and the wand issues you
  propose.
