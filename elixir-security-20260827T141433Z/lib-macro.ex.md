# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/macro.ex
<!-- sha256: 30d130be4957f46ab72c0524afa2b73c2237b66985c2555b001b0fbdfeab27bf -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:42:06Z_

I have enough evidence. The walkers grow the BEAM process stack on the heap (500k-deep processed fine → O(input) memory, no amplification), `to_string/1` escapes hostile literals, `inspect_atom` quotes/escapes correctly, and alias-expansion atom creation is bounded and inherent to compilation. The one concrete sink is the atom-table exhaustion in `generate_arguments`. Here is the report.

---

# Security review — `lib/elixir/lib/macro.ex`

## Summary
`Macro` is an AST-manipulation library with no file/network/shell/eval sinks; the only externally-influenced dangerous primitive is unbounded atom creation in `generate_arguments/2` and `generate_unique_arguments/2`, which can crash the entire VM.

## Security

#### `CSCAN-macro-generate-arguments-atom-exhaustion`
- **Finding ID:** `CSCAN-macro-generate-arguments-atom-exhaustion`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/macro.ex:530` (private `generate_arguments/3`), reachable via `generate_arguments/2` (`:409`) and `generate_unique_arguments/2` (`:523`).
- **Class:** Atom creation (CWE-770)
- **Issue:** `generate_arguments(amount, context)` executes `for id <- 1..amount, do: fun.(String.to_unsafe_atom("arg" <> Integer.to_string(id)), context)`. Each iteration mints a brand-new atom `arg<id>` with `String.to_unsafe_atom/1`, so the number of atoms created scales linearly with the caller-supplied `amount`. `amount` is typed `pos_integer` — a large value is *within* the documented contract, not a violation of it. The atom table is never garbage-collected and is globally bounded (default `atom_limit` ≈ 1,048,576); once it fills, the **whole node aborts**, not just the calling process.
- **Impact / threat:** A single call with a large count is a whole-VM denial of service. The realistic trust boundary is compile-time: these helpers exist to write macros/code-generators, and a generator that sizes its output from a value read at build time (a schema/`@external_resource`/config file, a field count from a dependency's data) would forward that count straight into the sink, crashing `mix compile` on the build machine. The atoms are `arg1..argN`, so they are bounded and reused across calls — only a single oversized call is dangerous, not repeated small ones.
- **Evidence:** `verified`.
  - Atom growth tracks `amount`: `Macro.generate_arguments(100_000, Elixir)` → `atoms before=18795 after=118793 delta=99998`, against `atom_limit=1048576`.
  - Whole-VM crash under a lowered limit (`ELIXIR_ERL_OPTIONS="+t 65536"`): `Macro.generate_arguments(200_000, Elixir)` printed `no more index entries in atom_tab (max=65536)` / `Crash dump is being written to: erl_crash.dump` / `exit=1` — the node died, not just the process.
- **Suggested fix:** Guard `amount` against an explicit sane upper bound (raise `ArgumentError` above, say, a few thousand) before the `for` comprehension, or document that `amount` must be a trusted, small constant. The current `String.to_unsafe_atom` on an unbounded caller-scaled count is the defect; capping the count removes it.

### Ruled out
- **Deep-recursion DoS in the AST walkers** (`prewalk/postwalk/traverse/2..4`, `path/2`, `prewalker/postwalker`, `validate/1`, `quoted_literal?/1`, `expand_literals/2,3`, deprecated `to_string/2`): *verified* not a distinct finding. BEAM grows each process's stack on its heap, so a 500,000-deep nested term passed to `Macro.prewalk`, `Macro.validate`, and `Macro.quoted_literal?` all returned `OK` with the process still alive; cost is O(input) memory with no fixed-depth crash and no super-linear amplification over just holding the input. Only the calling process is affected.
- **`compile_apply/4`** (`:1922`): documented as "apply a `mod`/`fun`/`args` at compile-time" — it *is* `Kernel.apply/3` with a tracer. A caller-controlled MFA is the function's contract, exactly like `apply/3`; not untrusted-data-reaching-a-sink.
- **`expand_once/2` / `expand/2`** (`:2008`, `:2327`): documented to expand macros and aliases "as if by the Elixir compiler" (running macro expanders, invoking tracers). Alias `concat` uses `binary_to_atom` (`elixir_aliases.erl:175`), so expansion creates module atoms — but this is inherent to compilation, bounded by the number of alias occurrences in the input, and equivalent to what compiling the same source already does.
- **`unescape_string/1,2`** (`:1245`, `:1291`): thin delegation to `:elixir_interpolation.unescape_string`; escape-sequence/bidi handling lives in that Erlang module, not here. It is a documented sigil-support utility, not an identifier-security boundary.
- **`to_string/1`** (`:1331`): *verified* it escapes hostile content — `Macro.to_string({:foo, [], [:"a\"; System.halt()  #", ...]})` produced `foo(:"a\"; System.halt()  #", ...)`, i.e. the payload stays inside an escaped atom literal (uses `Code.quoted_to_algebra`). No code injection into generated output. The deprecated `to_string/2` is display/debug only.
- **`inspect_atom/3` / `classify_atom/1` / `classify_with_tokenizer`** (`:2504`, `:2578`, `:2705`): operate on already-existing atoms for *display* format only. NFC/mixed-script/confusable enforcement is upstream in the tokenizer; here non-NFC/non-identifier atoms are classified `:other` and *quoted/escaped* when printed (`:"with spaces"`, `"\"a\\\"b\""` verified). Misclassification would be a formatting bug, not an acceptance bypass.
- **`postwalker/1` ref sentinel** (`:1170`): marks visited nodes with a fresh `make_ref()` the caller cannot forge in advance, so a crafted `{ref, node}` term in the input cannot collide with the sentinel.
- **`struct_info!/2` / `struct!/2`** (`:1035`, `:993`): compile-time struct loading via `:elixir_map`; `module` must already be an existing atom and loading a compiled BEAM module runs no arbitrary code beyond documented `@on_load` behavior.
- **`underscore/1` / `camelize/1`** (`:2380`, `:2444`): pure string transforms; they create no atoms and evaluate nothing. Documented as not for general-purpose use.

## Risks / caveats
- **Reachability of the one finding is the soft part, not the mechanism.** The VM crash is verified; what I cannot verify from this file alone is whether any real caller forwards an *untrusted* `amount` into `generate_arguments`/`generate_unique_arguments`. If none does, residual risk is a self-inflicted footgun; the compile-time code-generator path is the one that turns it into a genuine build-machine DoS, which is why it is reported rather than ruled out. I rated it Low conservatively — a reviewer who accepts the build-time-count vector may reasonably treat it as Medium.
- **Delegated logic is out of scope.** Several functions (`escape/2`, `unescape_string/*`, `struct_info!/2`, `expand_*`) hand off to `:elixir_quote`, `:elixir_interpolation`, `:elixir_map`, `:elixir_aliases`, `Code.Identifier`. Bidi/Trojan-Source, `\u`/`\x` escape validation, and identifier NFC/mixed-script rejection are enforced in those modules (tokenizer/interpolation), not in `macro.ex`; this file is a downstream caller, so those standards obligations were checked as "not enforced here" rather than as gaps.
- The `to_string/1` round-trip comparison returned `false`, but *verified* that this is metadata-only (reparsing adds `:line`); the escaped literal content itself round-trips faithfully, so no injection.

