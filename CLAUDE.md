# wmd

WMD (World of Many Doors, aka Wand MUD) is a MudOS-style MUD written in Wand.
The design doc is https://claude.ai/artifact/7LBnJra7BUMF4eHaaDFEqv. Read it
with the docs tools before you change the design.

## Rules

- Everything is Wand. The driver, the mudlib, the tests and every tool are
  `.wand` files. Do not write Python, shell scripts or a Makefile.
- When Wand cannot do something, do not use another language. Write it in
  Wand if you can. If the gap is general, propose a wand change.
- `wand t` typechecks. `wand s` runs the `test_*.wand` files. `wand f`
  formats. Run all three before you commit.
- A test file goes in a `test/` directory beside the code it tests, as
  `driver/test/test_engine.wand` for `driver/engine.wand`.
- `wand = ...` in `wand.pkg` is the oldest wand that wmd accepts. Change it
  only when wmd starts to use a newer feature.
