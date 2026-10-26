# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/pattern.ex
<!-- sha256: a3db1a42293c4699098e4880c7e961a427975dcfcfbbf141c64db52bcb2bd512 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:03:30Z_

# Security Review — `lib/elixir/lib/module/types/pattern.ex`

## 1. Summary
`Module.Types.Pattern` is an internal (`@moduledoc false`) type-checker pass that walks already-parsed pattern/guard AST during module compilation; it contains no code-execution, atom-creation, filesystem, network, or deserialization sinks, and no security-relevant defect reachable through a documented public API.

## 2. Security

`No findings`.

Rationale (trust boundary and reachability): every function here is reached only from `Module.Types.Expr`/`Module.Types.Of`, which are driven by `Module.Types.infer/7` — a pass that runs **during module compilation, after macro expansion**. Reaching this file therefore already implies arbitrary compile-time code execution, which the folder threat model places explicitly out of scope ("`eval_*`/compilation is documented to run code"). Tools that handle untrusted input by *parsing only* (`Code.string_to_quoted/2`, formatters, language servers) never invoke this pass. No sink in this file crosses a trust boundary that compilation itself does not already cross wide open.

### Ruled out

- **Atom exhaustion (CWE-770)** — *Basis: verified.* A search of the file for `to_atom`/`binary_to_atom`/`list_to_atom`/`Module.concat` returns zero matches. Every `atom(...)`, `integer()`, `binary()`, etc. is a `Module.Types.Descr` **type-descriptor constructor**, and the only atoms passed to `atom([atom])` (e.g. line 631) are literals already interned by the tokenizer/parser — no new atom is derived from external bytes. `make_ref/0` (lines 415, 1369) creates garbage-collected references, not atoms.
- **Code execution / eval / deserialization** — *Basis: verified.* Same file-wide search: no `Code.eval*`, `Code.compile*`, `apply(`, `:erlang.binary_to_term`, `EEx`. The module only builds and compares abstract type descriptors.
- **File / path / command / archive / network / crypto / regex sinks** — *Basis: verified.* File-wide search for `File.`, `Path.`, `System.cmd/shell`, `:os.cmd`, `:erl_tar`, `:zip`, `Regex`, network calls: zero matches. None of these sink families exist here.
- **Template / interpolation into a downstream interpreter** — *Basis: reasoned (code-structural).* `format_diagnostic/1` and `badpattern/2` (lines 1471–1842) interpolate source-derived text via `expr_to_string`/`to_quoted_string`/`indent` into developer-facing heredocs returned as `%{message: ...}`. There is no SQL/shell/EEx/markup interpreter downstream — the value is printed as a compiler diagnostic. Any terminal-escape re-quoting concern would live in the shared `Helpers` formatter, not in this file, and is only reachable by compiling the offending source.
- **Resource consumption / unbounded recursion (DoS)** — *Basis: reasoned + reachability.* The recursive walks (`of_pattern`→`of_tuple`/`of_list`, `of_pattern_tree`, `unpack_op`, `reconstruct_lists_member`) are bounded by AST size; the `of_changed_deps` worklist (lines 105–161) terminates because its `seen` set grows monotonically and edge count is bounded by version-pairs (O(V²)). Deeply nested source is bounded first by the tokenizer/parser, and any blowup here is strictly dominated by the compile-time code execution already required to reach the pass. Not a boundary this file defends.
- **`Enum.reduce/2` empty-collection crash** (lines 565, 960, 966, 1134) — *Basis: reasoned (traced every construction site).* Each reduce over a possibly-empty accumulator was checked: `{:intersection, entries}` is only built with a non-empty `dynamic` seed `[{:var, version}]` (lines 681, 694–698) or a 2-element list (line 766); the `of_list`/`of_guard` reduces run only on the `is_descr`/non-empty branches. No reachable empty-list path. Even if one existed, it would raise only inside the caller's compiling process — internal, not a security boundary.
- **Confusable / UTS-39 / NFC identifier checks** — *Basis: verified (out of file).* Those checks live in the tokenizer (`elixir_tokenizer.erl`) and `String.Tokenizer*`, not in this type-checker pass; nothing here weakens or bypasses them.

## 3. Risks / caveats
- **Verification performed:** confirmed the toolchain is built from this tree (`Elixir 1.21.0-dev`, OTP 29); confirmed the caller graph (`Module.Types.Expr`/`Of` → `Module.Types.infer/7`) so the pass is compile-time only; ran a file-wide search for the full dangerous-sink surface (0 matches). Ruled-out items marked *reasoned* were established by reading/tracing the code, not by executing adversarial inputs — I did not build a hostile module to drive stack/CPU exhaustion, because the reachability argument (compilation ⇒ code execution already granted) makes such a finding out of scope regardless of outcome.
- **Scope boundary:** this review covers only `pattern.ex`. Type-descriptor set operations it calls (`opt_intersection`, `opt_union`, `subtype?` in `Module.Types.Descr`) and diagnostic formatting (`Module.Types.Helpers`) are separate files; set-theoretic type operations have known worst-case blowup, but that is not this file's code.
- **Residual risk:** if a future caller ever invoked these exported (`@moduledoc false`) functions on attacker AST *without* full compilation, the DoS reasoning above would need re-checking — but no such documented public entry point exists today.

