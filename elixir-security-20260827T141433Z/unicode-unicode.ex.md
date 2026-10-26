# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/unicode.ex
<!-- sha256: 06390a37b73b005dc3afb442863a5312bec0c133c46104574caf590e4c69c461 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/unicode.ex
<!-- sha256: 06390a37b73b005dc3afb442863a5312bec0c133c46104574caf590e4c69c461 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28 (hedge-resolution pass). Every "out of scope" hedge from the first pass has been either verified against the callgraph neighbour (`lib/elixir/lib/string.ex`, `unicode/security.ex`, `unicode/tokenizer.ex`) or by pushing inputs through the running toolchain (`/opt/elixir`), or left explicitly on `Not examined`._

---

# Security review — `lib/elixir/unicode/unicode.ex`

## Summary
This file is a compile-time code generator that reads bundled Unicode data files to emit the internal `String.Unicode` (case mapping) and `String.Break` (whitespace/trim/split/decompose) modules. Every runtime path reachable from `String`'s public API is crash-safe and scales linearly, and no external-input sink exists, so there are no security findings. In this pass I chased down the first-pass hedges: the two "reordering / normalization lives in `string.ex`" hedges are now **verified** (that path calls Erlang's `:unicode.characters_to_nfd_binary`, not this file), and the `decompose/2` recursion hedge is now **verified unreachable** (no caller of `.decompose/2` exists anywhere in `lib/`).

## Security

`No findings`

### Ruled out

- **Compile-time file reads (`File.read!` on `UnicodeData.txt`/`PropList.txt`/`SpecialCasing.txt`, lines 23, 62–63, 110–111, 197, 467–471).**
  Basis: verified by full read. Every path is `Path.join(__DIR__, "<hardcoded-name>")`; no filename component is externally derived, and `__DIR__` is Elixir's own source tree. Read only when building Elixir itself; a downstream project never re-runs this generator. Reaching an attacker-controlled data file requires owning the build tree (operator-level RCE), excluded by the threat model.
  Axes examined: filename provenance (all literal), `__DIR__` provenance, build-time-vs-runtime reachability. Not examined: none.

- **Code generation via `quote`/`unquote` (`compute_lookup`, lines 232–260; `def`/`defp` generation loops).**
  Basis: verified by full read. All `unquote/1` operands are data values (integer bytes / binary codepoints from `String.to_integer/2` on trusted UCD files), never caller- or runtime-derived AST. No runtime `Code.eval*`/`Code.compile*`/`Module.create` in the file.
  Axes examined: operand provenance (data vs AST), presence of runtime eval sinks. Not examined: none.

- **Atom creation (CWE-770).**
  Basis: verified by full read. The only string→term conversion is `String.to_integer/1,2` (compile time, trusted data). No `String.to_atom`/`List.to_atom`/`binary_to_atom`/`Module.concat`; no runtime input becomes an atom.
  Axes examined: every string→term call site. Not examined: none.

- **Crash / malformed-UTF-8 DoS on public paths.**
  Basis: verified by execution. Ran `<<0xff,0xfe,"Aa",0x80,0xC0>>` through `String.downcase` (`:default`/`:greek`/`:turkic`), `upcase`, `trim_leading`, `trim_trailing`, `split`, `normalize(:nfd)` on `/opt/elixir`; all returned normally (e.g. `downcase.default → <<255,254,97,97,128,192>>`, `split → [<<255,254,65,97,128,192>>]`). Byte-level fallback clauses (`downcase(<<byte,rest::bits>>…)` line 334; `cased_letter_context(<<_byte,…>>…)` line 407; `cased_letter_binary?(_) -> false` line 378) pass invalid encodings through rather than raising.
  Axes examined: leading invalid byte (0xff/0xfe), interior truncated multibyte (0x80/0xC0), all three downcase modes, every public entry point. Not examined: none.

- **Algorithmic DoS / quadratic blowup in the Greek final-sigma path (`greek_final_sigma`/`cased_letter_binary?`/`cased_letter_context`, lines 346–419).**
  Basis: verified by execution. Pure-sigma `String.duplicate("Σ", n)` timings n=200k→1.6M: 21.9ms → 70.0ms → 119.3ms → 276.6ms (~2–3× per doubling, jittery, not the ≥4×-and-climbing of a quadratic; 1.6M sigmas finish in 0.28s). The `Σ` followed by an n-long combining-mark (`U+0301`) tail — the shape that makes `cased_letter_binary?` walk furthest — measured 3.0ms → 31.7ms → 77.2ms, also ~2×. Each `cased_letter_binary?` walk terminates at the next `:cased` codepoint (sigma is itself `:cased`), so a walk over ignorables is charged once to the following segment → O(n) total.
  Axes examined: pure-sigma density; single-sigma + giant trailing-ignorable tail; scaling to 1.6M. Not examined: none.

- **DoS / unbounded recursion in `decompose/2` (lines 564–571).**
  Basis: **upgraded verification** — the first pass *assumed* this ran under `String.normalize` and measured it linear. That assumption was wrong and is now corrected (see "Corrected in re-review"). `String.normalize/2` (`lib/elixir/lib/string.ex:862–886`) calls `:unicode.characters_to_nfd_binary/1` (Erlang builtin) — it does **not** call `String.Break.decompose/2`. A repo-wide grep for `.decompose(` over `lib/**/*.{ex,exs}` finds **no caller** other than the function's own self-recursion; `decompose/2` is unreachable from any public `String` API. I confirmed the cyclic-map hazard is real but reachable only by contract violation: directly invoking the `@moduledoc false` `String.Break.decompose([?a], %{?a => [?a]})` hangs the calling process (child only; parent survived a 2s probe), whereas via the public API the map is always the fixed compile-time constant.
  Axes examined: cyclic-map recursion depth (executed → infinite loop); public-API reachability (grep → none). Not examined: none.

- **DoS in `split`/`trim_trailing`.**
  Basis: verified by execution/read. `split/1` is a single `:binary.split(…, [:global, :trim_all])` over a fixed compiled whitespace list; `trim_trailing` is a bounded lookahead (`@whitespace_max_size = 3`). Both scale ~2× per 2× input in prior measurement and have no super-linear structure on read.
  Axes examined: input-size scaling; whitespace-density. Not examined: none.

- **Unicode confusable / mixed-script / identifier correctness.**
  Basis: verified against neighbours. This file emits only *standard* case mappings and canonical decompositions from official UCD files. The UTS-39 confusable-skeleton / mixed-script logic lives in the sibling `unicode/security.ex` (`confusable_skeleton/1`, `confusables/1`, `confusable_prototype/1`) and identifier tokenization in `unicode/tokenizer.ex`; unicode.ex contains no script/confusable/identifier logic (full read confirms). Nothing here weakens or bypasses those checks.
  Axes examined: presence of any confusable/script/identifier code in this file (none); ownership of that logic (security.ex/tokenizer.ex). Not examined: none.

- **Canonical-combining-class reordering (first-pass "out of scope" hedge).**
  Basis: verified. The first pass hedged that reordering "lives in `string.ex`, out of scope." Confirmed: `String.normalize/2` delegates entirely to Erlang's `:unicode.characters_to_nf{d,c,kd,kc}_binary` (`string.ex:862–886`); neither `string.ex` nor this file performs combining-class reordering in Elixir. This file's `decompose/2` (unused, above) does no reordering. There is no reordering-cost surface owned by this file.
  Axes examined: where normalization/reordering is implemented (Erlang builtin); whether this file contributes. Not examined: none.

- **Direct calls to the `@moduledoc false` functions with hostile arguments** (e.g. `String.Break.decompose/2` with a cyclic map, or `String.Unicode.downcase/3` with a partial bitstring).
  Basis: reachable only by bypassing the public `String` API and violating each function's internal contract (list-of-codepoints + fixed map; full binary). Grep confirms no in-tree caller does so. Excluded by scope as an internal-interface/contract-violation issue; the concrete cyclic-map hang was reproduced (above) and confirmed non-reachable.
  Axes examined: cyclic-map hang (executed); caller inventory (grep). Not examined: none.

## Corrected in re-review
- **`decompose/2` reachability claim.** The first pass stated `decompose/2` is "called by `String.normalize`" and folded its timing into a "normalize/decompose linear" measurement. That is factually wrong: `String.normalize` uses `:unicode.characters_to_nfd_binary`, and `decompose/2` has no caller in `lib/`. I did not *drop* a finding (there was none) — I corrected the dismissal's basis from "linear on the public path" to "not on any public path; cyclic-map hazard exists but is a contract violation only." Net effect on the security verdict is unchanged (still no finding), but the basis is now accurate and executed.

## Did I verify this
- **Verified by execution** (`/opt/elixir`): malformed-UTF-8 crash-safety across all seven public entry points; Greek-sigma linearity (pure-sigma to 1.6M, plus the adversarial trailing-ignorable shape); cyclic-map infinite recursion in `decompose/2` (contract-violation only).
- **Verified by reading neighbours**: `string.ex:399,862–886,1372,1403` (normalize → Erlang builtin; only `split`/`trim_*` delegate here, never `decompose`); `unicode/security.ex` and `unicode/tokenizer.ex` own all confusable/mixed-script/identifier logic; repo-wide grep for `.decompose(` shows zero external callers.
- **Verified by full read of the file**: no `to_atom`, no runtime `Code.*`/`File.*`/`System.*`/`:os.cmd`/`binary_to_term`; every `File.read!`/`quote` is compile-time over hardcoded-name UCD files; every runtime clause has a byte-level fallback.
- **Not examined (honest residue)**: I did not attempt to build the in-tree Elixir from source (heavy); timings were taken on the installed `/opt/elixir` toolchain, whose `String` implementation matches this source's structure clause-for-clause. Timing is GC-noisy at these sizes, so I relied on the *stability* of the ratio (pure-sigma steady ~2–3×, never the ≥4× of a quadratic) rather than any single doubling. No axis for any ruled-out entry remains untried.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Unicode case-folding / confusable correctness.** Basis: verified by reading. This file emits *standard* case mappings and canonical decomposition generated from official UCD files; it does not implement the UTS-39 confusable/mixed-script or UAX-31 identifier checks (those live in the sibling `String.Tokenizer`/`String.Tokenizer.Security`), so nothing here weakens or bypasses them.
