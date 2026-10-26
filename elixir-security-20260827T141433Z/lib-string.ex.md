# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/string.ex
<!-- sha256: 916f87574682502d24477eff173a23ce690f3d5a9c9f53d10f51b44da12b7297 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:43:19Z_

All ruled-out mechanics now verified (0 crashes / 0 mismatches across the fuzzers; `byte_slice` offset math holds on arbitrary bytes + wild offsets). I have three verified, reachable findings. Writing the report.

# Security review — `lib/elixir/lib/string.ex`

## 1. Summary

The `String` module is memory-safe in its index/offset arithmetic, but several documented public functions that are explicitly specified to tolerate arbitrary/invalid-UTF-8 binaries can be driven to **quadratic CPU blowup** or **hard process crashes** by tiny hostile inputs, giving an attacker cheap denial-of-service against any service that runs these functions on untrusted bytes.

## 2. Security

#### `CSCAN-string-invalid-utf8-quadratic`
- **Finding ID:** `CSCAN-string-invalid-utf8-quadratic`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `string.ex:865` (`normalize/2`, all four forms 862–888); same pattern in `intersperse_bin/3` (1744–1756, used by `replace/4` with `""`), `intersperse_fun/3` (1758–1770), `slice_every/4` (2624–2638)
- **Class:** Resource consumption (quadratic blowup)
- **Issue:** The invalid-byte recovery branches rebuild the result by **prepending** to the recursive call: `good <> <<head>> <> normalize(rest, form)`. Because the growing accumulator is the *right* operand, each of the N recursion frames re-copies the entire suffix, so total work is O(N²) on input that repeatedly yields the `{:error, …}` path (i.e. invalid UTF-8). The valid-input path is linear, and `reverse/1` is spared only because it happens to append on the left (which the runtime optimizes).
- **Impact / threat:** `String.normalize/2` is routinely applied to untrusted text (usernames, search terms, request bodies) before comparison/storage, and its own docs invite invalid input ("Invalid Unicode codepoints are skipped"). A single ~400 KB invalid-UTF-8 body burns ~10 s of CPU (12 s for `:nfd`); a realistic mostly-text payload with scattered invalid bytes still costs ~2 s at 400 KB, and `String.replace(sub, "", ".")` hit 3.4 s at only 200 KB. Cost scales quadratically, so a few MB pegs a scheduler for minutes. Repeated requests exhaust all schedulers → node-wide unavailability. Trust boundary: untrusted bytes → linear-looking String call.
- **Evidence:** `verified`.
  - `String.normalize(copy(<<0x80>>,N), :nfc)`: N=25k→22.5 ms, 50k→59 ms, 100k→227 ms, 200k→1373 ms (≈4–6× per doubling).
  - 400 KB single input: `nfc` **10039 ms**, `nfd` **11770 ms**; mixed `"abc"<>0x80` ×100k (400 KB) **2083 ms**.
  - `String.replace(copy(<<0x80>>,N),"",".")`: 50k→92 ms, 100k→417 ms, 200k→**3427 ms**. `slice_every` 50k→23 ms → 200k→266 ms. Contrast: `normalize` on *valid* ASCII 200k→800k = 4.5→29.5 ms (linear).
- **Suggested fix:** Accumulate the good chunks / stray bytes into an iolist and call `IO.iodata_to_binary/1` once at the end (as the non-error paths already do), instead of `<>`-prepending the recursive result. This makes all four `normalize` forms and the `intersperse_*`/`slice_every` recoveries linear.

#### `CSCAN-string-grapheme-gc-crash`
- **Finding ID:** `CSCAN-string-grapheme-gc-crash`
- **Severity:** Medium
- **Confidence:** high
- **Location:** shared `:unicode_util.gc/1` sink — `do_graphemes/1` (2203–2209), `length/2` (2339–2345), `next_grapheme/1` (2229–2237), `first/1` (2283–2289), `do_reverse/2` (1820–1831), `slice_every/4`, `capitalize/2`, etc.
- **Class:** Resource consumption (crash / DoS via decoder)
- **Issue:** A 3-byte input consisting of an Extended_Pictographic Latin-1 char (`©` U+00A9 = `<<0xC2,0xA9>>`, or `®` U+00AE) followed by almost any trailing byte makes `:unicode_util.gc/1` raise `ArgumentError` (`:erlang.++([169 | <<128>>], [<<128>>])` inside `gc_ext_pict/3`, `unicode_util.erl:1130`). Root cause is upstream OTP (stdlib 8.0.1 / OTP 29), but `string.ex` reaches it through its documented public API and its `case :unicode_util.gc(...)` sites only handle `[gc|rest]`/`[]`/`{:error,…}` — not a raise. The module documents that it works with invalid UTF-8 ("Self-synchronization"), so this crash contradicts its stated robustness.
- **Impact / threat:** `String.length/1`, `graphemes/1`, `reverse/1`, `slice/2`, `at/2`, `capitalize/1`, `equivalent?/2`, `next_grapheme/1`, `first/1` (and everything built on grapheme iteration) crash on a 3-byte payload. Any endpoint that calls one of these on untrusted binary input can be forced to raise per request (500s, error-log flooding; if the call sits in a long-lived `GenServer`/`Task` handling the data, that process is killed and restarted). BEAM isolates the crash to one process, so it is availability-only, not RCE. Trust boundary: untrusted bytes → any grapheme-based String call.
- **Evidence:** `verified`. `String.length(<<0xC2,0xA9,0x80>>)` → `ArgumentError` (trace: `:unicode_util.gc_ext_pict/3` → `string.ex:2340`). Brute force: `<<0xC2,0xA9,X>>`/`<<0xC2,0xAE,X>>` crash for 256 of 512 trailing bytes; 2-byte `<<0xC2,0xA9>>` returns 1 (fine). Raw OTP: `unicode_util:gc(<<16#C2,16#A9,16#80>>)` → `{'EXIT',{badarg, … '++'([169|<<128>>],[<<128>>]) … gc_ext_pict}}`, whereas `gc(<<16#C2,16#A9>>)` = `[169|<<>>]`. Cross-API check: `length/graphemes/reverse/slice/at/capitalize/equivalent?` all `CRASH`, `split/1` (via `String.Break`) `ok`.
- **Suggested fix:** Primary fix is upstream in `:unicode_util.gc_ext_pict/3`. In-repo, either pin/require an OTP version without the defect, or defensively treat a `gc/1` raise on a leading byte as the existing `{:error, <<byte, rest>>}` recovery so the module keeps its documented "invalid data is self-synchronized" guarantee instead of crashing.

#### `CSCAN-string-normalize-raises-invalid`
- **Finding ID:** `CSCAN-string-normalize-raises-invalid`
- **Severity:** Low
- **Confidence:** high
- **Location:** `string.ex:870` and siblings — `normalize/2` for `:nfc` (869–874), `:nfd` (862–867), `:nfkd` (876–881), `:nfkc` (883–888); reached also via `equivalent?/2` (805–807)
- **Class:** Resource consumption (crash) / Validation (documented contract violated)
- **Issue:** `normalize/2` is written assuming `:unicode.characters_to_nfX_binary/1` returns either a binary or `{:error, Good, Rest}` (it pattern-matches exactly those). For some invalid inputs (e.g. `<<0xE2,0x97,0x80>>` = U+25C0 ◀, Extended_Pictographic, followed by a stray continuation byte) the underlying OTP routine instead **raises** `badarg` ("not valid character data"), which `string.ex` does not catch. This breaks the function's documented promise that "Invalid Unicode codepoints are skipped and the remaining of the string is converted" — it raises rather than skipping.
- **Impact / threat:** `String.normalize/2` and `String.equivalent?/2` crash on crafted invalid-UTF-8 input. Where normalization is used as a pre-comparison/canonicalization step on untrusted data, the attacker turns it into a per-request crash (DoS) instead of the graceful skip the docs guarantee. Isolated per-process (availability only). Trust boundary: untrusted bytes → `normalize`/`equivalent?`.
- **Evidence:** `verified`. Fuzz (100k inputs/form): `normalize` raises on 13–20 cases per form. Minimal: `String.normalize(<<0xE2,0x97,0x80,0x80>>, :nfc)` → `ArgumentError`, trace `:unicode.characters_to_nfc_binary(...)` at `unicode.erl:1369` → `string.ex:870`; `<<0xE2,0x97,0x80,0x41>>` (valid trailing) → `ok`. Raw OTP: `unicode:characters_to_nfc_binary(<<16#E2,16#97,16#80,16#92>>)` → `{'EXIT',{badarg,…}}`.
- **Suggested fix:** Wrap the `:unicode.characters_to_nfX_binary/1` call so a `badarg` raise is funneled into the same single-byte skip-and-continue recovery already used for the `{:error, …}` return (or fix upstream `:unicode`), so the documented "skips invalid codepoints" behavior actually holds. At minimum, correct the docs to state that invalid input may raise.

### Ruled out

- **`to_atom/1` (3009–3013) and `to_unsafe_atom/1` (3038–3042) atom-table exhaustion** — Out of scope: these are the documented "sharp tool" primitives (analogous to `Code.eval_string`). `to_atom/1` is deprecated pointing at `to_existing_atom/1`; `to_unsafe_atom/1` is named "unsafe" and carries a prominent "atoms are not garbage-collected … should not be an untrusted value" warning; the safe `to_existing_atom/1,2` are provided. The definitions are correct; the danger is caller misuse, which the rules exclude. Basis: read code + docs.
- **`valid?/2` `:fast_ascii` returning wrong answer** — Differential fuzz of 20k random byte strings: `:fast_ascii` vs `:default` produced **0 mismatches**. Basis: verified.
- **`length/2` SWAR fast path (2347–2365) miscounting graphemes** — Differential fuzz (20k ASCII-heavy-incl-`\r` + 20k random) vs a naïve `:unicode_util.gc` walk: **0 mismatches** (the crash surfaced during this fuzz is the unrelated upstream `gc` bug, not a SWAR miscount). Basis: verified.
- **`byte_slice/3` `invalid_suffix` off-by-one → out-of-bounds `binary_part`** — Invariant `last + truncated == initial_last` bounds the return by `min(size, byte_size)`; fuzz of 50k arbitrary-byte strings with wild start/size offsets: **0 crashes**. Basis: verified.
- **`slice/3`, `at/2`, `split_at/2` index arithmetic (negative/oversized indices)** — Fuzz 50k each with wild indices/ranges (ASCII, to isolate from the `gc` bug): **0 crashes / no out-of-bounds**. Basis: verified.
- **`printable?/2` passing bidi/C1-adjacent controls (0xA0–0xD7FF) as printable (345–366)** — Working as documented: it is a printability predicate, not a Trojan-Source/bidi filter; that defense lives in the tokenizer, not here. Not this function's contract. Basis: read code + docs.
- **`equivalent?/2` treating distinct byte sequences as equal** — Intended behavior (canonical NFD equivalence per its docs); this is the correct semantics for its purpose, not a confusable weakness. Basis: read code + docs.
- **`pad_*/3`, `duplicate/2` large-allocation** — Size is an explicit caller-supplied `count`, not attacker-smuggled through a decoder; documented behavior. Not an untrusted-parse sink. Basis: read code.

## 3. Risks / caveats

- **Upstream attribution:** The two crash findings (`grapheme-gc-crash`, `normalize-raises-invalid`) are rooted in OTP `:unicode_util`/`:unicode` (stdlib 8.0.1 / OTP 29), reproduced directly against raw Erlang. They are reported here because they are trivially reachable through this file's documented public API on in-contract input (a `binary()` that the module documents as tolerating invalid UTF-8), which the review scope explicitly includes as decoder-on-untrusted-input bugs. The primary fix likely belongs upstream; behavior may differ on other OTP releases (I verified only the OTP 29 build shipped with this source). The quadratic finding (`invalid-utf8-quadratic`) is a genuine in-file code-structure defect, fixable here.
- **Blast radius is availability-only.** All three are DoS (CPU exhaustion or single-process crash); none is RCE, memory disclosure, or a confusable/normalization *bypass*. BEAM process isolation limits crashes to the handling process, though the quadratic CPU/GC pressure and repeatable crashes still degrade a node.
- **Not exhaustively fuzzed:** I concentrated fuzzing on validators, index math, and the Unicode decode paths. The `String.Unicode`/`String.Break` delegated modules (upcase/downcase/greek/turkic, whitespace trimming) are separate files and out of scope here; I did not audit their internals, only that this file dispatches to them correctly.
- I did not find any code-execution, atom-injection-beyond-documented, path, command, deserialization, or crypto sinks in this file — it contains none.

