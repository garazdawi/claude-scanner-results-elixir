# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/info.ex
<!-- sha256: 680f792c54eb43415982ef9bcfc6c1dda61847b1ccc39140abcd6fab16de9c20 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:10:24Z_

## Summary
The module is almost entirely robust; the one genuine defect is that `i/1` on a module compiled in memory (e.g. one defined at the IEx prompt) prints a confusing `"[]"` for "Module bytecode".

## Bugfixes

#### `CSCAN-info-code-which-empty-charlist`
- **Finding ID** — `CSCAN-info-code-which-empty-charlist`
- **Severity** — Low
- **Confidence** — high
- **Location** — `info.ex:149-154`, `IEx.Info.Atom.module_object_file/1` (the anonymous function passed to `default_or_apply/2`)
- **Class** — Missing clause / non-total function (silent wrong return)
- **Impact** — When `i(Mod)` is called on a module that was loaded from memory rather than a `.beam` file, the "Module bytecode" line displays the literal string `"[]"` instead of a meaningful value. This is a very common IEx scenario: any module defined at the prompt (`defmodule Foo do … end`) or via `Code.compile_string/2` is memory-loaded, so `:code.which/1` returns `[]` (an empty charlist). The `[_ | _] = path` clause only matches non-empty charlists, and the fall-through clause — written `atom -> inspect(atom)`, clearly intended for atom results like `:preloaded`/`:cover_compiled`/`:non_existing` — instead catches `[]` (its variable `atom` has no `is_atom/1` guard) and renders `inspect([]) == "[]"`. The user sees a meaningless `"[]"` where the other no-file cases produce a readable token like `":preloaded"`.
- **Suggested fix** — Handle the empty-charlist case explicitly, e.g. add a clause `[] -> "no value found"` (or a message such as `"in memory"`) before the catch-all, or guard the last clause with `when is_atom(atom)` and let a leading `[] ->` clause cover it.
- **Threat note** — Not attacker-reachable; a pure correctness/display defect. Reached through the documented public path `IEx.Helpers.i/1` → `IEx.Info.info/1` when inspecting a memory-loaded module.
- **Evidence** — `verified`. `[{mod,_}] = Code.compile_string("defmodule ReplMod do def f, do: 1 end")` then extracting the `"Module bytecode"` entry of `IEx.Info.info(mod)` yields `"[]"` (command output above: `bytecode-for-repl-defined-module: "[]"`), while `:code.which(:erlang)` returns `:preloaded` and renders as `":preloaded"`. Confirmed `:code.which(mod) == []` for the in-memory module.

### Ruled out
- **`Any` fallback only handles `%module{}` structs** — I verified the 11 built-in protocol types (Atom, BitString, Float, Function, Integer, List, Map, PID, Port, Reference, Tuple) all have explicit impls, so the only values dispatched to `Any` are structs, which `%module{}` matches. A map whose `__struct__` is a non-atom dispatches to the `Map` impl, not `Any` (protocol dispatch requires an atom `__struct__`), so no crash. Not a bug.
- **BitString `cond` has no `true ->` catch-all** — the four branches (`is_binary`+printable / `is_binary`+valid / `is_binary` / `is_bitstring`) are exhaustive for every bitstring; `is_bitstring` is true for all inputs reaching this impl. Verified with `<<255,255>>`, `<<1::3>>`, `<<7>>`, empty binary — all handled.
- **`info_non_printable_string`/`info_binary` could get `nil` from `find_first_codepoint`** — each branch is only entered when the whole binary fails the corresponding `String.printable?`/`String.valid?` check, which guarantees at least one offending codepoint exists, so `Enum.find` never returns `nil`. Verified with `<<0>>` and `<<0xC3>>`.
- **PID/Port use `:erpc.call` even for local, non-distributed terms** — verified `i(self())` works on a `:nonode@nohost` node (erpc executes locally); dead process → `undefined` → `{"Alive", false}`; unreachable remote node → caught → `{"Alive", false}`. Behaves as designed.
- **`process_name([])` / `links([])`** — confirmed the list-form `process_info(pid, [:registered_name])` yields `[]` for unregistered processes, correctly mapped to `"not registered"`. Not a bug.
- **`raw_inspect/1` for Date/Time/DateTime/NaiveDateTime/Regex/Range** — ran each; `Inspect.Any.inspect(value, %Inspect.Opts{})` returns a formattable doc and the `case` tolerates both the bare-doc and `{doc, opts}` return shapes. No crash, correct raw struct output.
- **`@spec info/1` returning `String.Chars.t()` values** — all returned `info` values (booleans, integers, strings, atoms) implement `String.Chars`. No contract violation observed.
- **`module?/1` → `info_module/1` for on-disk-but-unloaded modules** — calling `mod.module_info()` auto-loads the module; `:code.get_object_code` having succeeded guarantees a readable beam. No crash.

## Risks / caveats
- I could not exercise the true remote-node paths (`:erpc.call` to a live peer for `PID`/`Port`, and remote-shell dispatch), only the local and disconnected cases; the code appears correct by inspection but a distributed cluster was unavailable in the sandbox.
- The `--fnu` latin1 locale warning was worked around with `ELIXIR_ERL_OPTIONS="+fnu"`; all Unicode-sensitive branches (`String.printable?`/`valid?`/`codepoints`) were tested under that setting.
- The one reported finding is cosmetic (misleading display, no crash or data loss); I judged it worth reporting because REPL-defined modules are a routine `i/1` target and the intended fall-through clause plainly did not anticipate the empty-charlist result.

