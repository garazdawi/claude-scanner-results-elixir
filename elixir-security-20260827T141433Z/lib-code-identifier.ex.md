# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/identifier.ex
<!-- sha256: d22069cfae2bb54d738567e9b43a20ff2ff3cfb1ff0dd7c1b36411fb5e7575da -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:27:00Z_

I have a complete, verified picture. All leaking codepoints are category `Cf` (Format) — the exact invisible-character class this escape routine targets — while the neighboring Arabic *letter* U+0628 round-trips fine.

---

# Security review — `lib/elixir/lib/code/identifier.ex`

## Summary
The module is a small, largely pure identifier/escape helper; its one security-relevant routine (`escape/4`, used by `inspect/1`, `Macro.to_string/1`, and `Code.Normalizer`) correctly neutralizes bidi/zero-width Trojan-Source characters but omits the Unicode *Prepended_Concatenation_Mark* format-character block, so those invisible characters pass through un-escaped.

## Security

#### `CSCAN-identifier-pcm-escape-gap`
- **Finding ID:** `CSCAN-identifier-pcm-escape-gap`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/code/identifier.ex:145-151` (`escape_char/2` printable pass-through); fix belongs in the confusables clause at `:116-143`.
- **Class:** Unicode and text (invisible-format-character escaping) / Round-trip integrity.
- **Issue:** `escape_char/2` maintains an explicit allow-list-of-shame that escapes "confusing" invisible format characters — BOM, mathematical invisibles, the nine bidirectional Trojan-Source controls ("source of vulnerabilities"), interlinear annotations, zero-width joiners, zero-width/non-break spaces, and line/paragraph separators (lines 116–143). The Unicode **Prepended_Concatenation_Mark** characters — U+0600–U+0605, U+06DD, U+070F, U+0890, U+0891, U+08E2, U+110BD, U+110CD — are the same general category (`Cf`, Format) and are conspicuously absent (note U+061C, an adjacent Arabic format mark, *is* in the list). They therefore fall through to the printable pass-through clauses (`char in 0xA0..0xD7FF`, line 147, and `char in 0x10000..0x10FFFF`, line 149) and are emitted raw.
- **Impact / threat:** `String.printable?/1` returns `true` for these characters, so `inspect/1` on an untrusted binary takes the `Identifier.escape` path (`inspect.ex:294`) and copies the raw format character into its output. Two consequences, both reachable through documented public APIs on data crossing a network/file/dependency boundary:
  1. *Invisible-character injection into diagnostic output* — the same class of defect the escape list exists to prevent for bidi/zero-width. An attacker who gets a `Cf` mark into a value that is later `inspect`ed lands an invisible, display-affecting character in logs/consoles/error messages that a human reviewer cannot see.
  2. *Round-trip breakage* — a Prepended_Concatenation_Mark immediately before the closing delimiter is absorbed with it by the tokenizer's grapheme-cluster scanning, so the emitted literal no longer re-parses. `inspect/1` and `Macro.to_string/1` output is no longer valid Elixir, which corrupts any pipeline that stringifies an AST holding such a value and recompiles it. This fails *safe* — the effect is a `missing terminator` tokenizer error, not silent code execution or an attacker-chosen semantic change — which is why this is Low rather than higher.
- **Evidence:** `verified`.
  - `inspect(<<0x0600::utf8>>)` → `<<34, 216, 128, 34>>` (raw mark between quotes); `Code.eval_string(that)` → `** missing terminator: "`. All 13 PCM codepoints reproduce (`String.printable? = true`, round-trip `false`); a PCM in mid-string round-trips fine and the neighboring Arabic *letter* U+0628 round-trips fine, isolating the trigger to the format-character class.
  - `unicode_util:category/1` confirms every leaking codepoint is `{other,format}` (`Cf`), matching the class the escape list targets; U+0628 is `{letter,other}`.
  - Bidi controls, by contrast, *are* handled: `inspect("a" <> <<0x202E::utf8>> <> "b")` → `"a\u202Eb"` for all nine Trojan-Source controls.
- **Suggested fix:** Add the Prepended_Concatenation_Mark codepoints to the escaped confusables clause (lines 116–143), e.g. `when char in 0x0600..0x0605 when char in [0x06DD, 0x070F, 0x0890, 0x0891, 0x08E2, 0x110BD, 0x110CD]`, so they are rendered as `\uXXXX`/`\u{…}` like the other invisible format characters. (The astral pair U+110BD/U+110CD also needs covering before the `0x10000..0x10FFFF` printable clause at line 149.)

### Ruled out
- **Atom creation in `extract_anonymous_fun_parent/1` (`:64-72`, `String.to_unsafe_atom/1`).** *Basis: verified.* The `is_atom/1` head means the input is already an interned atom; the derived parent name is a bounded substring (atoms cap at 255 codepoints, so no `SystemLimitError`), and to supply a *novel* input atom the caller must first intern it — no amplification (measured 1 new derived atom per 1 pre-existing input atom, and in the real callers `inspect(fun)`/stacktrace formatting the parent name is the already-interned enclosing function name). Not an atom-exhaustion vector; no crash on malformed shapes (`:"----"`, `:foo`, `:-` all return cleanly).
- **Quadratic/DoS blowup in `escape/5` binary accumulation (`:89-108`).** *Basis: verified.* The `<<acc::binary, …>>` accumulator hits the runtime's binary-append optimization; timing is linear across 50k→800k worst-case (all-escaping) inputs for both the valid-codepoint and invalid-byte paths (≈0.7→14 ms). No amplification.
- **String breakout / injection via `escape/4` for code generation.** *Basis: verified.* Delimiter (`"`), `#{`, and `\` are all escaped (`:89-95`, `escape_map/1`); 20,000 random-binary round-trips through `"` + escape + `"` were all faithful except the PCM class above (which fails *closed* as a parse error, not an injection). Invalid UTF-8 is surfaced as `\xHH` (`:106-108`), not silently dropped.
- **Compound guard on the public `escape/4` head (`:80-81`).** *Basis: verified.* Operator precedence makes `(is_integer(limit) and limit >= 0)` an independent disjunct, so a finite integer `limit` bypasses the `is_binary(binary)`/valid-`char` checks — but reaching that requires passing arguments that violate the `@spec` (`binary()`, `char() | nil`), and the result is a fail-safe `FunctionClauseError` in the private clauses, not unsafe behavior. Out of scope (spec-violating input) and non-exploitable.
- **`unary_op/1`, `binary_op/1` (`:18-58`).** Pure atom-keyed lookups returning constants or `:error`; no external-data sink.

## Risks / caveats
- The module is `@moduledoc false` (compiler-internal), so I scoped reachability through its **documented public callers** — `Kernel.inspect/2`, `Macro.to_string/1`, and `Code.Normalizer` — per the instruction not to exclude untrusted data that reaches a sink through a public API. If you consider the entire module strictly out-of-API, the finding degrades to a robustness note.
- The reported finding's root cause is split between `escape` (doesn't escape the marks) and the tokenizer's grapheme-cluster scanning (absorbs the following delimiter); I placed the fix in `escape` because that is where the sibling defenses (bidi/zero-width) already live and where round-trippable output is the contract. I did not audit the tokenizer itself here.
- "Invisible in a terminal" for the `Cf` marks is rendering-dependent; the *round-trip failure* and the *category/printable* facts are verified, the visual-spoofing consequence is asserted from their `Cf`/Prepended_Concatenation_Mark semantics.
- The PCM list is current as of the Unicode version bundled with this Elixir (1.21.0-dev / OTP 29); future Unicode additions to `Prepended_Concatenation_Mark` would need the same treatment, which argues for driving the escape set from the property rather than a hand-list.

