# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/string.ex
<!-- sha256: 916f87574682502d24477eff173a23ce690f3d5a9c9f53d10f51b44da12b7297 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:56:09Z_

Based on extensive review and randomized differential testing (hundreds of thousands of cases per function against independent oracles, plus manual verification of every doctest and the bitwise-guard arithmetic), I found no genuine bugs in this file.

## Summary
`String` (public API of `lib/elixir/lib/string.ex`) was reviewed function-by-function and fuzz-tested against independent oracles; every documented entry point behaves per its contract and no reachable defect was found.

## Bugfixes

No findings

## Risks / caveats
- I loaded and tested the module both from the installed `/opt/elixir` build (v1.21.0-dev) and from a fresh `elixirc` compile of the exact source under review (force-loaded via `:code.load_abs`); behavior was identical, confirming the runtime reflects this source.
- Coverage of `String.Unicode` / `String.Break` internals (the `defdelegate`/`String.Unicode.*` targets for `split/1`, `trim_*`, `upcase/downcase :default/:greek/:turkic`) was **black-box only** — those live in other files and are out of scope here; I verified the documented behavior at the `String` boundary via doctests, not their internals.
- `replace_invalid/2` was validated against a from-scratch implementation of the Unicode "maximal subpart" substitution policy; if the intended policy ever differed from that recommendation, my oracle would not detect it (but the policy matches the published doctests).

### Ruled out
- **`length/1` SWAR fast path (`skip_length`, `ascii_no_cr_swar?`, lines 2333–2365)** — the hand-rolled 56-bit CR/ASCII detection is the most error-prone code here. Fuzzed `String.length` against `Enum.count(String.graphemes/1)` over 40k+ random strings with CR/LF at every position and 8-byte-aligned windows: 0 mismatches. Basis: `verified`.
- **`slice/2` (ranges + steps) and `slice/3`** — fuzzed against a grapheme-list oracle over 80k random strings (ASCII, multibyte `é`, 4-byte `😀`) with negative/positive/out-of-range indices and steps 1–3: 0 mismatches. Basis: `verified`.
- **`byte_slice/3` (`invalid_prefix`/`invalid_suffix`)** — tested truncated-codepoint trimming at both ends including negative offsets and 4-byte emoji: all correct (my initial hand-computed expectations were wrong, not the code). Basis: `verified`.
- **`split/3` (`split_each`/`split_empty`/`do_splitter`, parts+trim arithmetic)** — fuzzed against an independent reference over 80k cases across `:infinity`/1/2/3/4 parts × trim × single/multi-char patterns: 0 mismatches. The `when is_binary(string)` guard on the `count==1` clause correctly prevents a `:nomatch` sentinel from leaking (my first oracle lacked it and produced false mismatches). Basis: `verified`.
- **`replace/4` (`do_replace`)** — fuzzed vs `:binary.replace` (global + single) over 30k cases: 0 mismatches. Basis: `verified`.
- **`replace_invalid/2` (all `do_replace_invalid` clauses + `replace_invalid_*` guards)** — the bitwise range constants (`32..863`/`896..1023`, `16..271`, `1024..17407`) were derived by hand and confirmed to be the exact UTF-8 non-overlong/non-surrogate ranges; fuzzed vs a maximal-subpart reference over 200k random byte strings: 0 mismatches. Basis: `verified`.
- **`valid?/2` (`:default` and `:fast_ascii`)** — fuzzed both against `:unicode.characters_to_binary` round-trip over 100k byte strings: 0 mismatches. Basis: `verified`.
- **`pad_leading/3`, `pad_trailing/3` (`build_filler` cycle math)** — traced the `div(count,size)+1` accounting by hand and fuzzed vs a `Stream.cycle` reference over 40k cases with binary and list padding: 0 mismatches. Basis: `verified`.
- **`bag_distance/2`** — the `diff1 - length1 + length2` branch was proven equal to the reverse multiset difference (identity `Σmax(c2−c1,0) = Σmax(c1−c2,0) + (len2−len1)`) and all doctests pass. Basis: `verified`.
- **`count/2`, `contains?/2`, `starts_with?/2`, `ends_with?/2`, `splitter/3`, `chunk/2`** — fuzzed against independent references / invariants (non-overlapping match count, `:binary.match`, prefix/suffix byte comparison, splitter≡split, chunk-join round-trip) over 60k cases each: 0 mismatches. Basis: `verified`.
- **Dead `[gc]`/`[gc, rest]` clauses in `capitalize/2`, `next_grapheme/1`, `do_last`, etc.** — inspected `:unicode_util.gc/1` return shapes; the tail is always a binary (`[cp | ""]`) or `[]`, so a proper 2-element list never occurs and `[gc | rest]` handles all real inputs. Defensive/unreachable, not a defect. Basis: `verified`.
- **`to_existing_atom/2` doctest raising the custom message** — confirmed `:unknown` already exists in the runtime, so `binary_to_existing_atom` reaches the `not in allowed_atoms` check and the documented `ArgumentError` message is produced. Basis: `verified`.
- **`normalize/2` (all four forms) and `equivalent?/2`** — verified with explicit-codepoint inputs (an earlier apparent NFD failure was a shell-encoding artifact of passing precomposed literals through `elixir -e`, not a code bug). Basis: `verified`.

