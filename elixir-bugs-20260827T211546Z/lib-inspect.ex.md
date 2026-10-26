# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/inspect.ex
<!-- sha256: 4c6984cf8a1f282d888dfb7476e823c3decac912498100effef6d81e114b2c8b -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/inspect.ex
<!-- sha256: 4c6984cf8a1f282d888dfb7476e823c3decac912498100effef6d81e114b2c8b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (second pass). File located at `/opt/elixir/lib/elixir/lib/inspect.ex`; sha256 confirmed identical to the recorded `4c6984c…b2c8b`. The scanner checkout under `…/elixir/lib/elixir` contains only `src/*.erl`; the `.ex` library sources are not vendored there, so the byte-identical installed copy (sha-verified) is the reviewed artifact._

# Bug-hunting review (re-review): `lib/elixir/lib/inspect.ex`

## Summary
The `Inspect` protocol implementations are behaviourally sound across every path I exercised. The one defect is a documentation-contract mismatch: the moduledoc promises alphabetical / `:only`-customizable struct field ordering, but the code emits struct-definition order and ignores `:only` order entirely — re-confirmed by execution this pass.

Two hedges from the first pass are addressed:
- The "outside this file's control" hedge about BEAM primitives (`:erlang.*_to_list`) was retired by execution — re-confirmed here.
- The one entry that remained `asserted` in the last pass — the `Function.info/1` key-presence axis "across every fun flavour" — I closed by **executing** seven distinct fun flavours through `Inspect.Function.inspect/2` (not by reading the `Function` module, which would not clear the axis). All pass; the residual untried axis is narrowed to the single arm I could not construct a live trigger for (`not is_atom(mod)`), which stays on `Not examined`.

No cross-file assumption in the prior report is left as a bare assertion: each is either verified by execution or explicitly carried as an open, un-constructible axis.

## Bugfixes

#### `CSCAN-inspect-struct-field-order-doc-mismatch`
- **Finding ID** — `CSCAN-inspect-struct-field-order-doc-mismatch`
- **Severity** — Low
- **Confidence** — high
- **Location** — `inspect.ex:33-34` and `:56` (moduledoc); behaviour realized in `Inspect.Any.inspect/2` (`:663-670`), the derived `for` comprehension (`:227-231`), and `Inspect.Map.inspect_as_struct/4` (`:471-474`).
- **Issue** — The moduledoc makes two ordering claims the code does not honor:
  1. Lines 33-34 show the default representation of `defstruct [:id, :name, :address]` as `%User{address: "Earth", id: 13, name: "Jane"}` (alphabetical), and line 56 states "customize the order of fields **(the default is alphabetical)**". The actual output is definition order `%User{id: 13, name: "Jane", address: "Earth"}`. No code path in this file sorts struct fields; both the non-derived (`Inspect.Any`, `:665`) and derived (`:228`) paths iterate `module.__info__(:struct)`, which is definition order (confirmed by execution below, incl. a struct whose definition order is unambiguously non-alphabetical: `[:zeta, :alpha, :mid, :beta]`).
  2. Line 56 says deriving can "customize the order of fields." The `:only` list is used only as a membership guard (`var!(field) in unquote(filtered_fields)`, `:199`), never as an ordering; `only: [:name, :id]` and `only: [:id, :name]` produce identical output. `filtered_fields` is even built from the alphabetically-sorted `fields` (`:167`, `:193-195`) but its order is discarded by the `__info__(:struct)`-driven comprehension.
- **Class** — Documentation mismatch.
- **Impact** — Readers relying on the documented example or the "alphabetical" / "customize order" contract get a different field order than shown, and cannot reorder fields via `:only` as the docs imply. If alphabetical ordering was the intended behaviour (the sorted `fields` and the alphabetical doc example both suggest it once was), this is a silently-shipped behavioural regression that would also break downstream doctests asserting the old order; otherwise the docs are simply stale. Either way the file's own contract is internally inconsistent.
- **Evidence** — `verified` (re-executed this pass, `ord.exs`):
```
%User{id: 13, name: "Jane", address: "Earth"}       # doc claims %User{address:..., id:..., name:...}
%Zzz{zeta: nil, alpha: nil, mid: nil, beta: nil}     # defstruct [:zeta,:alpha,:mid,:beta] -> definition order, non-alphabetical
#U2<id: 1, name: "Jane", ...>                        # @derive only: [:name, :id]
#U3<id: 1, name: "Jane", ...>                        # @derive only: [:id, :name]  -> identical to U2; :only order ignored
User struct order: [:id, :name, :address]            # __info__(:struct) is definition order
```
- **Suggested fix** — Decide which side is the contract. If definition order is intended, update line 34 to `%User{id: 13, name: "Jane", address: "Earth"}` and change line 56 to say the default is definition order and that `:only`/`:except` only filter (not reorder) fields. If alphabetical/customizable order is intended, restore it by iterating over the (sorted, `only`-ordered) `filtered_fields` instead of `module.__info__(:struct)` in the derived comprehension and by sorting in `Inspect.Any.inspect/2`.

## Risks / caveats
- I could not determine which side of the ordering mismatch is authoritative: this artifact is the installed library (no `.git`, no CHANGELOG, no test suite alongside it), so I cannot confirm whether the alphabetical behaviour was intentionally removed or regressed. The mismatch itself is verified by execution; the "which is wrong" attribution is my best reading of the sorted-`fields` code smell. This is an intent-attribution caveat, not an unverified cross-file assumption.
- All main data paths I exercised (bitstrings incl. non-byte-aligned, floats incl. negatives/boundaries, ranges incl. empty/descending, improper lists, keyword detection, all four `@derive` options, regex option translation, Erlang-backed PID/Port/Reference, and seven Function flavours) return correct output — re-executed this pass, see Ruled out.

### Ruled out

- **Erlang-backed impls `PID`/`Port`/`Reference` pattern matches (`:643-660`)** — First pass hedged this as "output shape I trusted … outside this file's control." Retired by execution and re-confirmed this pass. `:erlang.ref_to_list(make_ref())` → `#Ref<…>`, so `[?#, ?R, ?e, ?f] ++ rest` (`:657`) matches and yields `#Reference<…>`; `:erlang.pid_to_list(self())` → `<0.95.0>`, so `"#PID" <> …` (`:645`) yields `#PID<0.95.0>`; `:erlang.port_to_list(port)` → `~c"#Port<0.0>"`, used verbatim (`:651`) yielding `#Port<0.0>`. End-to-end confirmed: `inspect(self())` → `#PID<0.95.0>`, `inspect(make_ref())` → `#Reference<0.1613016395.4010803201.109903>`, `inspect(hd(Port.list()))` → `#Port<0.0>`. No user-controllable input changes these primitives' output format. `verified`. Axes examined: prefix-shape of `ref_to_list`/`pid_to_list`/`port_to_list`, badmatch risk at `:657`. Not examined: none.
- **`Function.inspect/2` env/type/module branches (`:589-641`)** — First pass left this `asserted` with an open axis "exhaustive `Function.info` key-presence across every fun flavour." Closed this pass by **execution** (`funs.exs`), pushing seven flavours through `inspect/1`: anonymous closure-with-env → `#Function<0.72466766 in file:funs.exs>` (compiler-module `__RELATIVE__` arm, `:606-608`); anonymous no-env → same arm; multi-arity anonymous → same arm; module-level anonymous → `#Function<1.… /1 in Foo.make_anon/0>` (`default_inspect`, `:618`); local capture `&bar/1` → `#Function<0.… /1 in Foo.bar>` (`default_inspect`); external capture `&Enum.map/2` → `&Enum.map/2` (external+`env==[]` arm, `:601`); `Function.capture(String,:upcase,1)` → `&String.upcase/1` (same arm). For each, `Keyword.keys(Function.info(fun))` was dumped: closures/local-captures/module-anons all carry `[:pid,:module,:new_index,:new_uniq,:index,:uniq,:name,:arity,:env,:type]`; external captures carry `[:module,:name,:arity,:env,:type]`. Every key the code reads positionally (`:module`,`:name`,`:arity`,`:type`,`:env`,`:new_index`,`:uniq`) is present on the flavour whose arm consumes it (external arm never calls `uniq/1`, so its absence of `:new_index`/`:uniq` is harmless). No crash, no `badmatch`, three of four `cond` arms exercised live. `verified` for the tried flavours. Axes examined: closure/no-env/multi-arity/module-anon/local-capture/external-capture/`Function.capture`; the external-`env==[]` arm, the compiler-`__RELATIVE__` arm, and the `default_inspect` arm; key-presence per flavour. Not examined: a live trigger for the first `cond` arm (`not is_atom(mod)`) — I could not construct a fun whose `Function.info[:module]` is a non-atom; that arm reads only `uniq(fun_info)` and `:arity`, both present on every flavour observed, but the arm itself stays untried. `asserted` for that single arm only; no defect observed.
- **`Regex` `translate_options/2` order sensitivity (`:561-572`)** — a non-canonical opts order returns `:error` and falls back to `Regex.compile!(...)`, a valid (if verbose) representation, not a crash or wrong value. `verified` (re-run): `~r/foo/`, `~r/foo/ims`, `~r/foo/u`, `~r{a/b}`→`~r/a\/b/` all inspect correctly. Axes examined: canonical/non-canonical opt lists, `/`-escaping via `normalize`. Not examined: none for the tried option set.
- **`Float.inspect` integer-valued branch (`:528`)** — `abs >= 1.0 and abs < 1.0e16 and trunc(float) == float` correctly routes `-3.0`→`"-3.0"`, `1.0e16`→`"1.0e16"`, `0.1`→`"0.1"`. `verified` (re-run). Axes examined: negative, lower boundary, upper boundary, sub-1.0. Not examined: none for the tried values.
- **`Range.inspect` empty/step clauses (`:695-715`)** — `1..0//1` (empty, step 1) correctly falls to the step-showing clause; `0..0` shows no step; `10..1//-1` correct. `verified` (re-run): `1..0//1`, `0..0`, `10..1//-1`. Axes examined: empty range w/ positive step, degenerate single-element, descending explicit step. Not examined: none for the tried values.
- **`each_bit/3` counter/limit (`:322-345`)** — the `counter == 0 → "..."` clause precedes the byte clauses so the limit short-circuits correctly; `:infinity` never decrements to 0; non-byte-aligned tails hit the `::size(n)` clause; empty bitstring → `<<>>`. `verified` (re-run): `<<1,2,3,4::size(3)>>`→`<<1, 2, 3, 4::size(3)>>`, `<<5::size(3)>>`→`<<5::size(3)>>`, `<<>>`→`""`(binary path). Axes examined: mixed byte + sub-byte tail, pure sub-byte, empty. Not examined: none for the tried values.
- **`@derive` `:only`/`:except`/`:optional`/`:all` filtering (`:169-222`)** — all four produce the documented *field set*; only the *order* claim is wrong (reported above). `verified` (re-run confirms field-set membership; the two `only` orderings collapse to identical output, which is the ordering defect, not a filtering defect). Axes examined: field membership under each option. Not examined: none for filtering; ordering is the reported finding.
- **Improper / keyword list detection (`:389-426`)** — `[1|2]`→`[1 | 2]`; proper keyword lists → `[a: 1, b: 2]`. `verified` (re-run). Axes examined: improper tail, proper keyword. Not examined: none for the tried values.

### Did I verify this
- **Struct field-order finding** — Yes, re-executed this pass (`ord.exs`): definition order emitted, alphabetical doc example contradicted by a non-alphabetical `defstruct`, the two `:only` orderings collapse to identical output, and `User.__info__(:struct)` dumped as `[:id, :name, :address]` (definition order).
- **BEAM-primitive shapes (first pass's "outside this file" hedge)** — Yes, executed `:erlang.ref_to_list/pid_to_list/port_to_list` directly and end-to-end `inspect/1`; each emitted shape satisfies the corresponding pattern match / concatenation. No `badmatch` under the installed OTP.
- **`Function.info` key-presence (first pass's remaining `asserted` axis)** — Yes, executed this pass across seven fun flavours; dumped `Keyword.keys/1` per flavour and confirmed every positionally-read key is present on the arm that consumes it. The only residual is the un-constructible `not is_atom(mod)` arm (see Ruled out); I did **not** substitute reading the `Function` module for this — the narrowing is executed evidence.
- **All other Ruled-out data paths** — Yes, re-ran `misc.exs`/`port.exs` this pass; outputs reproduced verbatim above.
- **Not verified by execution** — only the `not is_atom(mod)` `cond` arm of `Inspect.Function` (`:598`), for which I could not construct a fun with a non-atom module. It stays `asserted`; no defect observed and both fields it reads are present on every flavour tried.

### Dropped in re-review
- None. Every finding and dismissal from the first pass is retained. Two verification upgrades relative to the first pass: (1) the "outside this file's control" caveat about BEAM primitives remains a `verified` Ruled-out entry backed by executed `:erlang.*_to_list` output; (2) the `Function.inspect/2` entry moves from `asserted` (open axis: "every fun flavour") to `verified` for seven executed flavours, with the `Not examined` list narrowed by execution to the single un-constructible `not is_atom(mod)` arm.
