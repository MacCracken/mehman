# Getting started with mehman

## Build

```sh
cyrius deps                                            # resolve sibling deps
cyrius build programs/smoke.cyr build/mehman-smoke       # compile-link smoke
cyrius distlib                                         # produce dist/mehman.cyr
cyrius test                                            # run tests/*.tcyr
```

## Layout

- `src/main.cyr` — library module (header). Add domain modules in sibling `src/` files; `programs/smoke.cyr` proves the include chain compiles.
- `programs/smoke.cyr` — minimal end-to-end smoke. CI builds this on every push.
- `tests/mehman.tcyr` — test cases. Use `assert_eq` / `assert` and exit with `assert_summary()`.
- `dist/mehman.cyr` — single-file bundle produced by `cyrius distlib`. Consumers `include` this from their own `cyrius.cyml [deps.mehman] modules = ["dist/mehman.cyr"]`.

## Adding a feature

1. Edit `src/main.cyr` (or add a new module and `include` it).
2. Add a test case to `tests/mehman.tcyr`.
3. Run `cyrius test`.
4. `cyrius distlib` to regenerate the bundle.
5. Bump `VERSION` and add a CHANGELOG entry before tagging.

See [`../adr/template.md`](../adr/template.md) when a non-trivial design choice deserves an ADR.
