# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/typespec.ex
<!-- sha256: caa6f1e2bcc7ff79122bd8b63076e7d1893139fafde294a0147b9340e6cbf4d1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:32:21Z_

I have completed the review. My analysis is verified against the source-built Elixir 1.21.0-dev toolchain in the sandbox.

---

# Security Review — `lib/elixir/lib/kernel/typespec.ex`

## Summary
`Kernel.Typespec` is an internal (`@moduledoc false`) compiler module that translates already-parsed Elixir AST for `@type`/`@spec`/`@callback` attributes into Erlang typespec forms; every non-trivial entry point runs *only during module compilation*, and I found no sink that grants capability beyond the arbitrary compile-time code execution that compiling a module already implies.

## Security
No findings.

The file contains no external-string→atom conversion, no path/file/command/network/crypto/deserialization sinks, and no Unicode/identifier handling (that belongs to the tokenizer; here identifiers arrive as already-interned atoms). The only code-execution-shaped operations (`Macro.expand/2` for record/struct/alias resolution at lines 602, 647, 989, 999; `:elixir_aliases.concat/1` at line 991) are ordinary compile-time macro/alias expansion, reachable only by compiling a module — a context in which the compiled code (macros, `mix.exs`) already runs arbitrarily on the build machine.

### Ruled out

- **Reachability gate (foundational).** Merely parsing hostile source does **not** reach any logic in this file — only compiling a module does. Basis: `verified`. `Code.string_to_quoted("defmodule Z do\n@type t :: some_undefined_type()\nend")` returns `{:ok, _}` without error, while `Code.compile_string/1` on the identical source raises `Kernel.TypespecError` from this module. Thus the trust boundary to reach any sink here is "compile untrusted source," which is arbitrary compile-time code execution and is explicitly out of scope per the ground rules (a bug requiring the victim to compile/eval attacker code is not a finding about this repo).

- **Compile-time code execution via `Macro.expand/2` (records/structs/aliases, lines 602/647/989/999) and `@module_attribute.type` reads (line 777).** Basis: `verified` (compile-time reachability). These expand macros/aliases the developer already has in scope; `name` is guarded `is_atom`, and expansion is the same primitive compilation uses everywhere. A malicious dependency gains nothing it doesn't already have — a `defmacro`/`__using__` in the same module runs at compile time regardless (observed a same-module macro body executing during `Code.compile_string`). No additional file/exec/traversal primitive is unlocked by a typespec. Dominated by inherent compilation trust.

- **Atom-table exhaustion via alias concatenation (`:elixir_aliases.concat/1`, line 991).** Basis: `verified`. Alias segments come from parsed source and the count of distinct concatenated module atoms is bounded by source size; this is identical to normal alias resolution. Compiling a single trivial module already interns ~1987 atoms (the compiler itself), swamping any typespec-derived contribution. No unbounded external-bytes→atom path exists in this file (no `String.to_atom`/`binary_to_atom`/`Module.concat`; the only atom↔list use is `:erlang.atom_to_list/1` at line 1082, the safe direction).

- **Resource exhaustion / stack overflow via deeply nested typespecs (recursive `collect_union/1` line 1014, `typespec/4`, non-tail walks).** Basis: `verified`. Compiling `@type t :: 1 | 2 | … | 200000` (a ~1.75 MB right-nested union) completed successfully ("COMPILED OK") on the growable BEAM stack. The AST must first be built by the parser, which recurses to the same depth, so this file adds no amplification over parsing, and reaching it still requires compilation.

- **Deprecated public functions `beam_typedocs/1`, `beam_types/1`, `beam_specs/1`, `beam_callbacks/1` (lines 24–65) accepting a binary `module`.** Basis: `asserted` (by delegation). They forward to `Code.fetch_docs/1` / `Code.Typespec.fetch_*` with a caller-supplied module-or-path; any BEAM-file reading occurs in `Code.*`, not here, and reading a file the caller explicitly named is expected behavior, not a traversal introduced by this file.

- **Confusable / mixed-script / NFC identifier checks (folder Unicode obligations).** Not applicable to this file: it operates on post-tokenization AST where identifiers are already atoms; UAX #31/UTS #39/#55 enforcement lives in the tokenizer and `unicode/` tables, not here.

- **Dead / unreachable defensive gates** (`arity in 0..255`, `unit in 1..256`, `size >= 0`, reserved-signature and built-in-type rejections, `validate_range`). Basis: `asserted`. Each is reachable from a corresponding crafted typespec and rejects malformed input with a compile error; none is a security gate whose bypass grants a capability, and none was found unreachable.

## Risks / caveats
- **Scope boundary is the crux.** Because parsing does not reach this code and compilation does, the entire module sits behind the "you are compiling this code" boundary. If a downstream consumer ever drove `translate_typespecs_for_module/2` or `deftypespec/6` on attacker AST *without* full module compilation, the DoS reasoning above (non-tail recursion, alias-atom creation) would need re-evaluation; I found no such caller in this file, but I did not audit the whole compiler for one.
- **Minor, unrated:** `IO.warn`/`compile_error` messages interpolate source-derived atom names and `Macro.to_string` output (e.g. lines 261, 298, 344, 781). An atom containing control characters could inject into a build log/terminal, but only when compiling hostile source (which already runs code) and only into the developer's own build output — not treated as a finding.
- The three reproductions were run against `elixir 1.21.0-dev` built from the repo under review (Erlang/OTP 29); behavior could differ on other OTP versions, though the recursion/atom semantics exercised are stable across them.

