# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/of.ex
<!-- sha256: 372dffe737c24b0f87ebfa1a354891c7db290b86398d952a1a905ae195169beb -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:27:34Z_

## Summary

The file is the compiler-internal type-checking helper `Module.Types.Of`; its logic (map/struct/bitstring type inference, variable refinement) is internally consistent and its diagnostic-formatting paths run without crashing, but one exported function's `@doc` contract contradicts what the function actually returns.

## Bugfixes

#### `CSCAN-of-refine-body-var-doc-return`
- **Finding ID** — `CSCAN-of-refine-body-var-doc-return`
- **Severity** — Low
- **Confidence** — high
- **Location** — `of.ex:82-88` (`@doc` for `refine_body_var/5`), function bodies at `of.ex:96-154`
- **Issue** — The `@doc` states *"Returns `true` if there was a refinement, `false` otherwise."* Every return path of `refine_body_var/5` actually returns a `{type, context}` two-tuple (`of.ex:111,124,133,136,148,151`), never a boolean. The documented return contract does not match any code path.
- **Class** — Documentation mismatch
- **Impact** — A maintainer or caller trusting the `@doc` would write `if refine_body_var(...)` and always take the truthy branch (a non-`nil`/`false` tuple is always truthy), silently masking the intended "no refinement" case; anyone destructuring per the doc as a boolean gets a tuple. In-file callers already destructure the tuple (`{_, context} = refine_body_var(...)`, `of.ex:251`), so the code is self-consistent and only the documentation is wrong — but `@doc` is a stored, fetchable contract even under `@moduledoc false`.
- **Evidence** — `verified`. `Code.fetch_docs(Module.Types.Of)` returns for `refine_body_var/5` the last doc line `"Returns \`true\` if there was a refinement, \`false\` otherwise."`, while every clause of the function returns `{old_type|new_type|none(), context}`. Command: `elixir refine_probe.exs` → `refine_body_var/5: last doc line => "Returns \`true\` if there was a refinement, \`false\` otherwise."`
- **Suggested fix** — Replace the doc line with the real contract, e.g. *"Returns `{type, context}`, where `type` is the refined type (unchanged if no refinement occurred)."*

## Risks / caveats

- This module is entirely compiler-internal (`@moduledoc false`); there is no package-public API. I treated the exported functions as the review surface and confirmed the diagnostic paths are reachable during ordinary compilation (type warnings fire from user source), which is where untrusted-ish source text flows in.
- I verified by compilation that the `:badbinary` and `:badsize` `format_diagnostic/1` paths execute and render correctly (`elixir tc_test.exs`). I did **not** execute the `:badmap`/`:badkey`/`:badmodule`/`:unknown_struct*` formatters, though they are structurally analogous.
- I did not have a pristine upstream tree to diff against (the installed Elixir *is* the file under review), so I could not distinguish a planted deviation from a pre-existing upstream quirk; findings rest on in-file evidence only.

### Ruled out
- **`map_key_type`/`union_negated`/`closed_map` reduce (map inference, `of.ex:319-468`)** — traced the single/multiple/domain accumulator threading and the negated-key union against the inline worked examples; ordering via `Enum.reverse(single, domain)` and the cartesian distribution are consistent, and duplicate/negated/broad-domain keys are handled as documented. Basis: `asserted` (logic walk against the module's own comments); compiled maps with duplicate and negated keys did not crash the checker (`verified` no-crash).
- **Bitstring alignment (`bitstring/4`, `bitstring_segment`, `alignment_to_type`, `of.ex:564-653`)** — `alignment` sums bit widths and `rem(_,8)` maps to `binary()`/`bitstring_no_binary()`; unsized `binary`/`bits`, sized integer, and literal segments each produce the expected aligned/unaligned classification. Basis: `verified` (compiled `<<x::integer>>` on an atom → correct `:badbinary`; `<<1::size(x)>>` with binary `x` → correct `:badsize`).
- **Default-argument-in-middle heads (`declare_var/3`, `modules/8`)** — Elixir fills the skipped optional correctly when called with one fewer arg; parameter mapping is the intended one. Basis: `asserted` (language semantics).
- **`impl/2` calling `struct.__info__(:struct)` on a loaded non-Elixir atom** — could raise `UndefinedFunctionError`, but only reachable via internal protocol-dispatch types that are Elixir struct modules, not through any documented input path. Basis: `asserted` (caller-provided `for` is a struct type).
- **`all_same_conditional_vars?/1` / `reduce_conditional_vars/4` on `[]`** — non-total (would `FunctionClauseError` on empty list), but callers always supply ≥1 branch entry; not reachable with valid input. Basis: `asserted`.

