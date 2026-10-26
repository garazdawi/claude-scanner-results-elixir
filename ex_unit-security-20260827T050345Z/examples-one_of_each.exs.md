# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/examples/one_of_each.exs
<!-- sha256: ebcc589f78d4b2e98944531c5724f332413b2a527d89011cbbb332b66afd2ec0 -->

_Analysed on 2026-08-27T05:08:36Z_

This is a self-contained ExUnit example script. I've read the whole file. Let me work through the inventory.

- **Phase 1 inventory** — walking every sink class against the actual lines:

- **Code execution / compile-time execution**: The only "code" evaluated is the module's own literal source — `assert`/`refute`/`assert_raise`/`flunk` macros (lines 25–125) and literal funs in `Task.async`/`spawn_link` (lines 132, 175–192). No `Code.eval_*`, no `Code.compile_*`, no `unquote` of caller AST. Module attributes (`@one`, `@two`, `@long_data_1/2`, lines 15–19) are literals — no file/network reads at compile time.
- **Atom creation**: Every atom is a literal (`:ok`, `:foo`, `:no_message_there`, …). No `String.to_atom`/`Module.concat` on external bytes.
- **Command exec / file ops / archive / deserialisation / network / crypto / unicode / round-trip**: none present.
- **Template / interpolation**: All messages are literal strings (e.g. `flunk("we failed. totally")`, `Logger.debug("this will be logged")`). No `#{}` interpolation anywhere — no log/format injection.
- **Shared mutable state / concurrency**: `send(self(), …)` targets only the test's own mailbox; `spawn_link`/`Task.async` run literal closures with no external influence.
- **Resource consumption**: The blocking `receive do end` (line 134) and `1/val` (line 205) are intentional demonstrations of "exit" / "exception" failure styles, driven by hardcoded values, not external input.
- **Validation / dead defensive code**: no predicates, guards, or size checks exist.

**Phase 2 — trust boundaries**: This file is an example script under `lib/ex_unit/examples/`, not part of the compiled library. It exposes no public API (its only functions — `some_vars/2`, `blows_up/0`, `ignite/1` — are `defp`), takes no arguments, reads no files, and consumes no external data. Every value in the file is a hardcoded literal. There is no sink to drive with hostile input, so there is nothing behavioural to reproduce.

## Summary
`one_of_each.exs` is an ExUnit example script that defines a test module containing one intentionally-failing test of each style, used only to document how each failure renders; it has no public API, takes no external input, and contains no dangerous sinks — its security posture is benign.

## Security
No findings

### Ruled out
- **`spawn_link`/`Task.async`/blocking `receive` (lines 132–136, 175–192)** — run literal, hardcoded closures inside individual tests; no attacker-influenced value reaches them. These are deliberate demonstrations of exit/link failure rendering, not a concurrency defect.
- **`1 / val` with `val = 0` (lines 201–205)** and **`Access.fetch(:foo, :bar)` (line 166)** — intentional crashes to showcase exception/function-clause failure output; arguments are literals, not external input.
- **`Logger.debug("this will be logged")` (line 161)** and all `flunk`/assert messages — literal strings with no interpolation; no log-injection or format-string sink.
- **`ExUnit.start(seed: 0)` (line 6)** — fixed literal option; no external configuration influence.
- **`send(self(), …)` sequences (lines 70–73, 106, 111)** — self-directed messages with hardcoded payloads; not shared-state writes an attacker can influence.

## Risks / caveats
- I treated the file as a standalone example script with no caller and no documented public entry point; that is what it is (`lib/ex_unit/examples/*.exs`, run via `bin/elixir …`). If a downstream harness were to instead template or interpolate untrusted data into this file before execution, that generator — not this file — would be the review target.
- No reproduction was run because no sink exists to drive; the conclusion rests on the file being entirely composed of literals with no input surface, which a full read of all 207 lines confirms.
- Framework-level concerns (manifest `binary_to_term`, filter atom growth, capture/server races) noted in the folder threat model live in the ExUnit implementation modules, not in this example file, and are out of scope here.

