# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/security.ex
<!-- sha256: 2245b623ba598e36bf20861c2b13e706283b9099a97a997eec9a576512be9eef -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T14:40:27Z_

I've completed a thorough, execution-backed review. Every runtime path I could reach from the tokenizer (the realistic untrusted-source entry point) is crash-free, linear, and free of dangerous sinks; detection works end-to-end. Here is the report.

---

# Security Review — `lib/elixir/unicode/security.ex`

## Summary
`String.Tokenizer.Security` is a purely functional, advisory UTS#39 confusables-detection module whose runtime paths I verified to be crash-free, linear, and sink-free, with no meaningful security defect reachable from the compiler's untrusted-source entry point.

## Security

`No findings`

This file (`@moduledoc false`) is reached in production only via the tokenizer: `elixir_tokenizer.erl` calls `unicode_lint_warnings/1` (line 1913) and `confusable_skeleton/1` (line 1449) while tokenizing source. The untrusted input is therefore *source text*. The module has **no** code-execution, atom-creation, file/path/command/network, deserialization, or shared-mutable-state sink at runtime; its only side effect is emitting advisory **warnings**. The one `File.read!`/`Path.join` (lines 70–96) runs at **compile time of Elixir itself**, on a bundled file addressed by `__DIR__ <> "confusables.txt"` — no attacker-controlled component. I drove the reachable sinks directly and could not produce a crash, a super-linear blowup, or a detection bypass.

### Ruled out

- **Crash / `CaseClauseError` in `unbidify/1` (lines 166–192).** `String.Tokenizer.dir/1` returns exactly `:weak_number | :neutral | :ltr | :rtl` (tokenizer.ex:350–360); the `case` covers all four against both `part_dir` values (`^part_dir` + the two guarded switch clauses), and the final `case direction` covers both terminal directions. *Basis: verified* — fuzzed every length‑1..5 combination over representative identifier-valid codepoints of each `dir` class (`?a`, `?0`, `?_`, `0x05D0`, `0x0644`, `0xB7`, `0x05F3`): `total combos tested with crashes: 0`.
- **Crash in `confusable_skeleton/1` / `bidi_skeleton/1` on hostile/boundary codepoints.** *Basis: verified* — boundary set (noncharacter `U+FFFF`, max scalar `U+10FFFF`, NFD-decomposing `U+00C5`/`U+212B`, ligature `U+FB01`, combining mark after RTL, digit/rtl/digit) all returned normally, e.g. `U+212B → [65,778,1488]`. Token codepoints originate from valid UTF-8, so `:unicode.characters_to_nfd_list/1` never hits its `{error,…}` branch and `<<other::utf8>>` (line 104) never sees a surrogate/out-of-range value.
- **Quadratic / DoS resource consumption.** Each character joins exactly one direction "part" that is copied to `acc` once per direction switch, so total work is O(n). *Basis: verified* — `unbidify` on 200k chars = 5.6 ms, 400k chars = 12.8 ms (ratio 2.29× for 2× input, i.e. linear, not the ~4× of quadratic). Moreover identifiers reaching the check are bounded to ≤255 chars by Erlang's `list_to_atom` limit.
- **Detection false-negative (confusability bypass) from the check logic (lines 45–66).** The three `case` clauses partition all states — same name (`{_,_,^name}`), different name same skeleton (`:warn`), or absent (`_ ->` insert) — leaving no fall-through to silent acceptance. *Basis: verified* — end-to-end, `хо` (Cyrillic) vs `xo` (Latin) emits `confusable identifier: 'xo' looks like 'хо' on line 1…`; Cyrillic-vs-Latin `paypal` variants produce the identical skeleton `~c"paypal"`.
- **Inconsistent `confusable_prototype/1` return type (list of ints for table hits vs `binary` for the fallback, lines 98–104) causing skeleton mismatch.** Both shapes are valid `chardata`, and the trailing `:unicode.characters_to_nfd_list/1` (line 113) flattens either into the same flat codepoint list used as the map key. *Basis: verified* — the `paypal` test above mixes a table-hit prototype (`Cyrillic а → [0x61]`) with fallback binaries (`<<"p">>`, …) and still yields equal skeletons.
- **Atom exhaustion / compile-time execution.** No `*.to_atom`/`Module.concat`/eval anywhere; the only compile-time work is parsing the trusted bundled `confusables.txt`. *Basis: verified* (read whole file; no such call sites).
- **Unchecked quoted atoms (`atom_quoted`/`atom_unsafe`) absent from `@identifiers`.** These are *explicitly quoted* atoms (`:"…"`), outside UTS#39's unquoted-identifier scope; unquoted atoms (`:atom`) *are* checked. Intentional scoping, not a weakening. *Basis: asserted* (design intent per UTS#39 §1).

## Risks / caveats
- **Full UAX#9 correctness of `unbidify/1` is not exhaustively proven.** I confirmed the module's UAX#9 property-test cases and many hand-built RTL/neutral/number combinations, but cannot rule out a subtle mis-ordering for some exotic same-script RTL sequence. The blast radius is small: a mis-order would only cause a *missed advisory warning* (not an error, not code execution) for two **single-script** RTL identifiers — and cross-script confusables are independently hard-**errored** by the tokenizer's `mixed_script` check before this lint runs (observed: `pаypаl` → `invalid mixed-script identifier`). Residual security impact: low.
- **Warning messages interpolate the raw identifier text** (`"…'#{name}' looks like '#{previous_name}'…"`, lines 58–59), which can contain RTL *letters* when rendered to a terminal. This is inherent to reporting confusables and is mitigated by the per-codepoint `\uXXXX` breakdown in `dir_breakdown/1`. The nine bidi *control* characters cannot appear here — the tokenizer rejects them as non-identifier codepoints upstream.
- Scope note: the module is `@moduledoc false`. I treated its realistic reach as "the compiler tokenizing untrusted source"; findings reachable only by hand-calling the exported helpers with contract-violating arguments were considered out of scope per the review rules and, where testable, were shown robust anyway.

