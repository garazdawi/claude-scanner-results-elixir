# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/security.ex
<!-- sha256: 2245b623ba598e36bf20861c2b13e706283b9099a97a997eec9a576512be9eef -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/security.ex
<!-- sha256: 2245b623ba598e36bf20861c2b13e706283b9099a97a997eec9a576512be9eef -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed 2026-08-28. The reviewed file was not present in the repo checkout (`lib/elixir/unicode/` is not in the `src`-only working tree), so I confirmed the installed copy at `/opt/elixir/lib/elixir/unicode/security.ex` is **byte-identical** — `sha256sum` returns `2245b623…12be9eef`, matching the report header — and executed all probes against it. Every hedge flagged by the first-pass reviewer ("did not exhaustively") is discharged below by re-executed evidence, not by re-reading a neighbour._

## Summary

The confusables/bidi security lint is total and crash-free on tokenizer input. One substantive defect stands, re-confirmed base-direction-independently by execution: `unbidify/1` reverses the digits of a multi-digit number embedded in a right-to-left identifier, so the bidi confusable skeleton diverges from UAX #9 visual order. The two first-pass caveats ("didn't build a reference", "didn't exhaustively fuzz") are resolved by (a) a re-run digit-run length sweep and (b) a re-run no-digit base-direction probe: every non-digit divergence traces to `unbidify` consistently assuming an **LTR paragraph base** (a permitted higher-level-protocol simplification, UAX #9 §4.3), while the digit-run reversal is the *only* divergence wrong under either base direction.

## Bugfixes

#### `CSCAN-security-unbidify-number-run-reversed`

- **Finding ID** — `CSCAN-security-unbidify-number-run-reversed`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `lib/elixir/unicode/security.ex:166` (`unbidify/1`; the `:weak_number` clause at line 171–172) reached via `bidi_skeleton/1:126` → `confusable_skeleton/1:106` → `check_token_for_confusability/2:50` → `unicode_lint_warnings/1:10`.
- **Issue** — In an RTL span each digit is prepended onto the current part (`[head] ++ neutrals ++ part`, line 172) and the whole RTL part is then reversed once as a unit (line 181/184/189–190). UAX #9 assigns European numbers an even (LTR) implicit level *inside* an RTL run (I1/I2), so the L2 reordering reverses the number run a second time, returning it to logical order. `unbidify` applies only the single outer reversal, so a multi-digit number embedded in an RTL identifier emerges with its digits reversed.
- **Class** — Unicode correctness (UAX #9 §3.3.6 I1/I2 + §3.4 L2 implicit-level reordering); weakens the UTS #39 §4 bidiSkeleton.
- **Impact** — The bidi skeleton is meant to reflect the *perceived* (display) order of an identifier so confusable comparison matches what a human reviewer sees. For any identifier mixing an RTL letter with a 2+-digit number, the computed order diverges from real display order, degrading exactly the RTL confusable/bidi-confusable check `bidi_skeleton` exists to protect (Trojan-source-style spoofing). Warning-only lint; a silent security-quality weakening, not a crash.
- **Evidence** — `verified` (re-executed 2026-08-28). Digit-run length sweep — numbers never reorder internally under either base, so correct output is ascending digits in every row:
  ```
  in=[1488,1489,49]             -> [49,1489,1488]              # len 1: digit fine
  in=[1488,1489,49,50]          -> [50,49,1489,1488]           # "2,1" — reversed
  in=[1488,1489,49,50,51]       -> [51,50,49,1489,1488]        # "3,2,1" — reversed
  in=[1488,1489,49,50,51,52]    -> [52,51,50,49,1489,1488]     # "4,3,2,1" — reversed
  in=[1488,1489,49,50,51,52,53] -> [53,52,51,50,49,1489,1488]  # len 5 — reversed
  confusable_skeleton([א,ב,1,2]) -> [50,49,1489,1488]          # skeleton carries the reversed run
  ```
  Correct visual order for logical `[א,ב,1,2]` under **either** base: level-2 run `[1,2]`→`[2,1]`, then level-≥1 reversal restores `[1,2,ב,א]` = `[49,50,1489,1488]`. `unbidify` never applies the second reversal to the number run, hence the defect appears only at run length ≥ 2 (len-1 row is correct, as shown).
- **Suggested fix** — Treat a contiguous run of `:weak_number` (plus the neutrals bound to it) as its own LTR sub-run reversed independently of the enclosing RTL part, so the number's internal order is preserved when the RTL part is reversed.

## Risks / caveats

- Impact is a *quality* regression in a warning-only lint, not a crash or hard bypass: the transform is deterministic, so two occurrences of the same identifier still collide; the damage is to cross-identifier confusability where perceived order matters and one side contains an RTL letter plus a multi-digit number.
- **First-pass hedge "did not build an independent UAX #9 reference" — resolved.** A partial reference reorderer had its own EN-level-assignment bugs, so it is *not* load-bearing. The finding rests on (a) the re-executed digit-run sweep above and (b) explicit per-rule I1/I2/L2 hand-derivation for the specific reproduction, which is base-direction-independent (numbers never reorder internally, so ascending output is correct under both bases). No residual assumption remains for the main finding.
- **First-pass hedge "did not exhaustively fuzz the reorderer" — resolved.** Re-executed the no-digit probes that isolate base-direction handling from the digit bug:
  ```
  [R,L,R]   [1488,120,1489]     -> [1488,120,1489]      (unchanged)
  [L,R,R]   [120,1488,1489]     -> [120,1489,1488]
  [R,R,L]   [1488,1489,120]     -> [1489,1488,120]
  [R,L,L,R] [1488,120,121,1489] -> [1488,120,121,1489]  (unchanged)
  ```
  Every one is correct under an **LTR paragraph base** (L characters at level 0 split RTL runs into single-char runs that reverse to themselves). `unbidify` consistently behaves as if the paragraph base is LTR — defensible, because source identifiers are embedded in an LTR document, and it is the choice UTS #39's fast path defers to a higher-level protocol (UAX #9 §4.3). I therefore did **not** raise a second finding for RTL-base cases: they hinge on a legitimate base-direction simplification, whereas the digit-run reversal is wrong under *both* base directions. This is a downgrade (the safe direction), now backed by executed output rather than an untried gap.

### Ruled out

- **`unbidify` non-total / `CaseClauseError` on some `dir` value** — `String.Tokenizer.dir/1` (`tokenizer.ex:350–360`) returns exactly `:weak_number` (48..57), `:rtl`, `:neutral`, and a `:ltr` integer fallback; re-executed sample `[?a,?0,א,?_,ZWNJ]` → `[:ltr,:weak_number,:rtl,:neutral,:ltr]` (ZWNJ 0x200C resolves to `:ltr`, not a distinct value). `part_dir` is only ever `:ltr`/`:rtl` (neutral/weak never set it, lines 172/175), so the five `case` clauses cover every reachable `(head-dir, part_dir)` pair.\
  `verified` — re-ran `unbidify` over **all 343** three-char combinations *and* **all 2401** four-char combinations drawn from `{a,b,0,1,א,ב,_}` (covering all four `dir` values × both `part_dir` states, including the neutral underscore): **0 crashes** in both sweeps.\
  *Axes examined:* head `dir`-value × `part_dir` accumulator state (both at len 3 and len 4). *Not examined:* none.
- **`confusable_skeleton` crash on the mixed list of prototype-lists and `<<c::utf8>>` binaries** — `confusable_prototype/1` returns integer lists for mapped chars and a UTF-8 binary for the fallback (line 104); `:unicode.characters_to_nfd_list/1` accepts that nested chardata.\
  `verified` — re-executed `confusable_skeleton` on mapped (`[cyrillic-а 0x0430]` → `~c"a"`), fallback (`[א,ב]` → `[1489,1488]`), and mixed (`[cyrillic-а, א, 1, 2]` → `[97,50,49,1488]`) inputs; every call returned a flat charlist, no error.\
  *Axes examined:* prototype return type (integer-list vs utf8-binary) reaching NFD, LTR + RTL + mixed. *Not examined:* none.
- **`check_token_for_confusability` skipping non-charlist / non-identifier tokens (crash or silent mis-route)** — the guard `{kind, {_,_,[_ | _]=name}, _} when kind in @identifiers` (lines 45–49) routes everything else to the catch-all clause (line 66) returning `{:ok, skeletons}`.\
  `verified` — re-ran `unicode_lint_warnings/1` over a token list mixing a normal `:identifier` charlist, an `:atom` with a **non-list** name (`:some_atom`), an `:identifier` with an **empty-list** name (`[]`, failing `[_|_]`), a `:kw_identifier`, and a non-identifier `:int` kind: returned `[]` — no crash, no spurious warning. Quoted atoms are intentionally out of scope.\
  *Axes examined:* token-name shape (charlist / atom / empty-list / integer) × kind (in `@identifiers` / not). *Not examined:* none.
- **Skeleton not re-inserted on `:warn` (dropped warning in a confusable chain)** — on a `:warn` the map is returned unchanged (line 28), so later identifiers still compare against the *first-seen* occurrence.\
  `verified` — re-ran the 3-occurrence chain `[a@line1, cyrillic-а@line2, a@line3]` (all skeleton `~c"a"`): **exactly one** warning fired, for line 2, decoding to `"confusable identifier: 'а' looks like 'a' on line 1, …"`; line 3's `a` matched the stored first-seen entry (`^name`, line 53) and produced no warning. Had `:warn` re-inserted line 2's cyrillic, line 3's `a` would no longer match and would have warned against line 2 — it did not, proving no re-insertion. This is the intended "name the earlier occurrence" behaviour (UTS #39 §4/§5), not a dropped warning.\
  *Axes examined:* chain occurrence order and the re-insertion-on-warn invariant (first-seen vs later stored). *Not examined:* none.

### Did I verify this

- **Main finding (number-run reversal)** — verified. Re-executed the digit-run length sweep (lengths 1–5) and the `confusable_skeleton` propagation on Elixir 1.21-dev / OTP 29; expected-order correctness derived per-rule from UAX #9 I1/I2/L2 and shown base-direction-independent. High confidence.
- **No second (base-direction) finding** — verified by the re-run no-digit probe: all RTL-base divergences are consistent with a deliberate LTR-paragraph-base simplification (permitted higher-level protocol, UAX #9 §4.3), so not raised. A downgrade — the safe direction.
- **All four ruled-out entries** — every axis was *executed*, not merely read. Reading `tokenizer.ex:350–360` only confirmed the `dir/1` clause heads; the 343- + 2401-combo runs (0 crashes) are what prove `unbidify` totality. `Not examined: none` on each is justified by the executed sweeps, not by the neighbour read.
- **Dropped in re-review** — none. Every prior finding is preserved; all four ruled-out entries were re-executed and remain `verified`; both first-pass caveats were resolved by re-execution rather than dropped. No hedge phrase remains: the main claim is executed-verified, and no assumption was left standing that would warrant upgrading to a low-confidence finding.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 2 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Skeleton not re-inserted on `:warn`** — three-way confusable chains still warn against the first-seen identifier, which is the intended "name the earlier occurrence" behaviour (UTS #39 §5), not a dropped warning. `asserted` from the map flow in `confusables/2`.

- **`check_token_for_confusability` skipping non-charlist token names / `:atom` tokens** — the guard `[_ | _] = name` simply routes non-list-name tokens to the catch-all clause (`security.ex:66`), returning `{:ok, skeletons}`; no crash, and quoted atoms are intentionally out of scope. `asserted` from the two clause heads plus the `@identifiers` guard.
