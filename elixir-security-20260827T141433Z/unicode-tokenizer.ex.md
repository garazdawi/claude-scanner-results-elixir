# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/tokenizer.ex
<!-- sha256: 3612eb4fd3a00389146aa5018695eaa2ad171cffb8abb179f44bcc4776ff4f15 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/unicode/tokenizer.ex
<!-- sha256: 3612eb4fd3a00389146aa5018695eaa2ad171cffb8abb179f44bcc4776ff4f15 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T (hedge-resolution pass)_

## Summary

The identifier tokenizer contains a quadratic-time blowup in its non-NFC error-diagnostic path that lets a single crafted identifier in untrusted source make `Code.string_to_quoted` / `mix compile` burn minutes-to-hours of CPU. This pass re-verified the one hedge flagged by the first-pass reviewer ("outside this file", in the atom-exhaustion dismissal) by reading the caller module `elixir_tokenizer.erl`; that dismissal now rests on the caller's actual code and its 255-codepoint atom cap, not on an assumption. No prior finding was dropped; no new finding was upgraded (the hedge verified out).

## Security

#### `CSCAN-tokenizer-nfc-prefix-quadratic`
- **Finding ID:** `CSCAN-tokenizer-nfc-prefix-quadratic`
- **Severity:** High
- **Confidence:** high
- **Location:** `tokenizer.ex:542-550` (`invalid_original_prefix/3`, the `:unicode.characters_to_nfc_list(:lists.reverse(prefix))` at line 545), driven from `validate/2:484` and `invalid_normalized/3:533`.
- **Class:** Resource consumption (algorithmic-complexity DoS, CWE-407 / CWE-1333)
- **Issue:** When a Unicode identifier is **not** already in NFC form (`different? = true` at `validate/2:476`), `validate/2` calls `invalid_normalized/3` (line 484). If the NFC-normalized form does not re-tokenize cleanly as the same kind — most easily achieved when the NFC form is **mixed-script** — control reaches the `_other` clause (`invalid_normalized/3:533`), which calls `invalid_original_prefix(original_acc, acc)`. That function walks the identifier character by character and, for **every** position, reconstructs and NFC-normalizes the entire growing prefix (`:unicode.characters_to_nfc_list(:lists.reverse(prefix))`, line 545). This is Θ(n²) time (and allocation) in the identifier length `n`. The identifier length is attacker-controlled and effectively unbounded (bounded only by source-file size), and the diagnostic only refines an error message — it has no bearing on the accept/reject decision.
- **Impact / threat:** `String.Tokenizer.tokenize/1` is invoked by the Erlang tokenizer for every identifier (confirmed: called at `elixir_tokenizer.erl:1070` inside `is_unnecessary_quote/2` and, for the main identifier path, via `tokenize_identifier` which dispatches to `(Scope#elixir_tokenizer.identifier_tokenizer):tokenize(...)`). The blowup is reachable through the primary documented public API for parsing untrusted Elixir source — `Code.string_to_quoted/1,2` — and therefore through everything built on it: `mix compile` / `mix deps.compile` of a hostile dependency (the build machine is the target), `mix format`, language servers (ElixirLS), doc/linter tooling (Credo), and any service that parses user-submitted snippets. A single identifier turns a trivially small file into a multi-second-to-multi-hour hang of the parsing process at 100% CPU. Measured: a 48 KB source with one crafted identifier parses in **17.8 s**, versus **42.6 ms** for a benign identifier of the same length (~420× amplification); because the cost is quadratic, a few-hundred-KB identifier reaches minutes and a few-MB identifier reaches hours. The trust boundary crossed is untrusted source text (dependency / build input / user submission) → the compiler front-end.
- **Evidence:** `verified`.
  - Quadratic scaling of `String.Tokenizer.tokenize/1` on a non-NFC, mixed-script identifier (`n` × decomposed `"e"+U+0301` then one Cyrillic `а`):
    ```
    n=2000  len=4001   time=280.4ms
    n=4000  len=8001   time=1063.5ms
    n=8000  len=16001  time=4245.6ms
    n=16000 len=32001  time=17792.6ms   # doubling input → ~4× time
    ```
  - Same length in NFC (benign) form is linear and fast: `len=16001 time=7.5ms`.
  - Reachable end-to-end via the public parser and the same asymmetry:
    ```
    Code.string_to_quoted: source_bytes=48006  time=17791.6ms   # one crafted identifier
    benign NFC same length: source_bytes=32006 time=42.6ms
    ```
  - Trigger confirmed to route through the diagnostic path (returns `unexpected_token`, not the direct `mixed_script` error): `String.Tokenizer.tokenize([101, 769, 1072]) => {:error, {:unexpected_token, [101, 769, 1072]}}`.
- **Note on the 255-codepoint cap:** the caller (`unsafe_to_atom/4`, `elixir_tokenizer.erl:1077-1086`) rejects identifiers longer than 255 with "atom length must be less than system limit", but that guard runs **after** `String.Tokenizer.tokenize/1` has already returned. The quadratic work happens *inside* `tokenize/1` on the full charlist before any length check downstream applies, so the 255 cap does not bound this path. (Confirmed by reading the caller: `tokenize_identifier` calls `tokenize/1` first; `unsafe_to_atom` is only reached on the success branch and after the full token is produced.)
- **Suggested fix:** Make the diagnostic linear or bound it. `invalid_original_prefix/3` only refines the error span, so the smallest safe change is to guard it behind an identifier-length cap (e.g., for identifiers longer than a few hundred characters, return the whole reversed token as the invalid prefix instead of scanning) — this removes the amplification with no effect on which identifiers are accepted or rejected. A more complete fix avoids the per-step full-prefix `:unicode.characters_to_nfc_list(:lists.reverse(prefix))` by advancing an incrementally-maintained normalized prefix (NFC only recombines within a bounded trailing combining run, so a small look-back window suffices) rather than re-normalizing from the start on every character.

### Ruled out

- **Compile-time `File.read!` / macro code generation (lines 35-137, 151-261, etc.)** — `Path.join(__DIR__, "…")` reads fixed Unicode data files that ship inside the source tree and are consumed only while Elixir itself is built; no path component or content is reachable from any runtime public API. An attacker able to alter those files already controls the build. *Basis: reasoning about data flow — the inputs are `__DIR__`-relative constants, not derived from `tokenize/1`/`dir/1` arguments.*

- **Atom-table exhaustion** — *(Hedge "outside this file" resolved by reading the caller.)* This file performs no `String.to_atom`/`binary_to_atom` at runtime; `tokenize/1` returns the identifier as a charlist (`:lists.reverse(acc)`, `tokenizer.ex:469`). I read the consuming module: the atom is created in `elixir_tokenizer.erl` by `unsafe_to_atom/4` via `binary_to_atom(Binary, utf8)` (`1123-1125`) / `list_to_atom(List)` (`1153-1155`) — confirming the creation site is the Erlang caller, not this file. That caller **hard-caps identifier length at 255** before creating any atom (`unsafe_to_atom/4` first clause, `1077-1086`: `byte_size(Part) > 255` or `length(Part) > 255` → `{error, "atom length must be less than system limit"}`), so each accepted identifier yields at most one bounded-size atom. Atom-table exhaustion by many *distinct* identifiers in a large file is a general property of the Elixir tokenizer independent of this file and is bounded per-atom; it is not introduced or amplified here. *Basis: verified — read caller `elixir_tokenizer.erl:1070, 1077-1167`; this file creates no atoms and the downstream creation is length-capped.*

- **Mixed-script / confusable bypass (`validate/2`, `chunks_single?/2`)** — the UTS-39 mixed-script rejection fires as intended: `tokenize(~c"aа")` (Latin `a` + Cyrillic `а`) returns `{:error, {:mixed_script, …}}` (verified). Chunked identifiers (`foo_bar` across scripts) are intentionally permitted per UTS-55. I found no input that gets a mixed-script identifier *accepted*. *Basis: verified rejection on the canonical confusable; no accepting path constructed.*

- **`dir/1` non-integer input** — the `def dir(i) when is_integer(i), do: :ltr` catch-all (`tokenizer.ex:360`) makes it total for integers; a non-integer argument raises `FunctionClauseError`, but that violates the documented codepoint contract and is caller misuse. *Basis: excluded per scope (contract violation).*

- **Silent NFC normalization of identifiers (`validate/2:474`)** — the tokenizer returns the NFC form of an identifier, so NFC and NFD spellings of the same name converge to one atom. This is the safe direction for confusability (it collapses, not splits, look-alikes) and is Elixir's documented behavior. *Basis: reasoning — normalization reduces, not introduces, byte-vs-appearance divergence.*

## Risks / caveats

- **Non-mixed-script triggers not exhaustively mapped.** I verified the quadratic path via the reliable non-NFC + mixed-script trigger. Other inputs that make the NFC form fail to re-tokenize cleanly (e.g., NFC producing a trailing non-identifier codepoint, hitting the `{:error, {:unexpected_token, …}}` branch at `invalid_normalized/3:530`) would reach the same O(n²) code; I did not enumerate them because one clean trigger is sufficient to establish the defect. The fix should target `invalid_original_prefix/3` regardless of which upstream branch reaches it.
- **Whole-script confusables are out of scope by design.** UTS-39 mixed-script detection does not catch an identifier composed *entirely* of one non-Latin script whose glyphs resemble Latin (e.g., an all-Cyrillic look-alike). That is an inherent limitation of the standard, not a defect in this file, so it is not reported.
- **Severity anchoring.** Impact is DoS only (no memory corruption or code execution) and the trigger size scales with source-file size rather than being a tiny constant payload; I rated it High rather than Critical for that reason, while weighting up the reachability through `mix compile` of a dependency (build-machine reach) per the codebase's severity guidance. Extrapolations beyond the measured 17.8 s / 48 KB point are quadratic projections, not separately timed.

## Did I verify this?

- **Quadratic NFC diagnostic finding** — **Verified (executed).** Timing measurements on `String.Tokenizer.tokenize/1` and end-to-end `Code.string_to_quoted` show the ~4×-per-doubling quadratic signature and the ~420× amplification vs. the benign NFC baseline. The trigger's route through the diagnostic (`unexpected_token`, not direct `mixed_script`) was confirmed by the returned tuple.
- **Reachability through the caller** — **Verified (read).** `elixir_tokenizer.erl` dispatches to `(identifier_tokenizer):tokenize(...)` per identifier (`1070`, and the main `tokenize_identifier` path), confirming `tokenize/1` runs on every identifier in untrusted source. The quadratic work precedes any downstream length check.
- **Atom-exhaustion dismissal (previously hedged "outside this file")** — **Verified (read).** Atom creation occurs in `elixir_tokenizer.erl:unsafe_to_atom/4` (`1123-1167`), not in this file, and is length-capped at 255 (`1077-1086`). The hedge is resolved: the claim is confirmed against the caller's code, so the entry stays ruled-out for this file with `verified` basis.
- **Mixed-script rejection** — **Verified (executed):** canonical confusable `~c"aа"` rejected.
- **Data-flow / `dir/1` / silent-normalization dismissals** — **Reasoned, not executed.** These rest on reading the code paths in this file; no untried axis was narrowed by reading, and none is upgraded (each is a scope/contract or safe-direction argument, not an unverified security claim).

### Dropped in re-review

None. Every prior finding and ruled-out entry is retained. The only change is that the atom-exhaustion dismissal's hedge ("outside this file") was verified by reading the caller `elixir_tokenizer.erl` rather than assumed, and its basis upgraded from "verified absence (grep of this file)" to "verified against the caller's creation site and 255-codepoint cap."
