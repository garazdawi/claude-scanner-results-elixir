# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/option_parser.ex
<!-- sha256: a1b789a76464cceab54b8beed828a87b628a6cd731d1be9c528a880f4505895f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:31:42Z_

## Summary

After a full-file review of `option_parser.ex`, driven by running the module's complete official test suite (52 tests + 35 doctests) plus targeted edge-case probes against the reviewed source, I found no genuine, reachable bug in the documented public API.

## Bugfixes

No findings.

Every candidate defect I enumerated was verified against the reviewed source (compiled standalone from `/Users/lukas/.../lib/option_parser.ex`) and behaved as documented. Concretely:

- The entire upstream test file `option_parser_test.exs` (`Result: 87 passed (35 doctests, 52 tests)`) passes against this exact source.
- All 35 published doctests execute and match.

## Ruled out

- **`allow_nonexistent_atoms: true` → `String.to_unsafe_atom/1` (line 855) is a nonexistent-function crash.** Reason: the function exists in this build; `verified` — `OptionParser.parse(["--foo-bar","baz"], switches: [], allow_nonexistent_atoms: true)` returned `{[foo_bar: "baz"], [], []}`.
- **`did_you_mean` threshold `score > 0.8` (line 895) too strict / wrong constant.** Reason: `verified` — the three suggestion tests (`--doc-bar`→`--docs-bar`, `--docs_bar`→`--docs-bar`, `--verbos`→`--verbose`) all pass, producing the exact expected messages.
- **`format_error/3` regex clause (lines 907–912) crashing or mis-formatting on invalid regex / `nil` type.** Reason: `verified` — `parse!(["--rx","["], strict: [rx: :regex])` produced `--rx : Invalid regular expression "[": missing terminating ] for character class at position 1`; alias-invalid `{option,value}` entries (`-l xyz`) correctly resolve the type via `get_type` (the alias key atom always exists because the caller passed it as an `aliases:` literal), yielding `Expected type integer, got "xyz"`.
- **`normalize_value`/`value_in_tail?` mis-classifying negative numbers vs unknown one-letter switches as the next value (lines 807–835).** Reason: `verified` — `parse(["--num","-5"], strict: [num: :integer])` → `{[num: -5], [], []}`; `parse(["--num","-x"], strict: [num: :integer])` → `{[], [], [{"--num", nil}, {"-x", nil}]}` (arg not consumed, matches upstream).
- **`expand_multiletter_alias` (lines 774–793) mis-expanding grouped aliases / `-nN` numeric shorthand.** Reason: `verified` — `-abc` with three boolean aliases expands to all three true; matches upstream (the `-n2=foo` double-`=` quirk is pre-existing upstream behavior on malformed input, not a defect in this file).
- **`to_argv/2` `:count` handling (lines 567–573) producing wrong argv.** Reason: `verified` — `to_argv([verbose: 3], switches: [verbose: :count])` → `["--verbose","--verbose","--verbose"]`; doctests for `:count` and boolean/`nil` cases pass.
- **`split/1` failing to raise on an unterminated quote or mishandling escapes (lines 595–631).** Reason: `verified` — `split(~s(foo "bar))` raised `argv string did not terminate properly, a " was opened but never closed`; escaped-quote/space doctests pass.
- **`--no-SWITCH=value` negation with a value silently accepted (lines 795–801).** Reason: `verified` — `parse(["--no-op=x"], switches: [op: :boolean])` → `{[], [], [{"--no-op", "x"}]}` (correctly invalid).
- **Underscore-switch rejection and "Did you mean" suggestion (lines 466, 883–902).** Reason: `verified` via the underscore test (`--docs_bar` → invalid + `Did you mean --docs-bar?`) passing.

## Risks / caveats

- I did not find and confirm an injected defect; this file passes its full published contract (doctests) and its complete official unit-test suite unchanged, which is strong evidence of correctness but is a suite-completeness claim, not a proof of absence.
- The reviewed source redefines the already-installed `OptionParser`; I compiled and loaded `option_parser.ex` directly (`Code.require_file`) so every result above exercises the reviewed file, not the prebuilt `/opt/elixir` beam.
- Scope was limited to `option_parser.ex` and its documented public API (`parse/2`, `parse!/2`, `parse_head/2`, `parse_head!/2`, `next/2`, `to_argv/2`, `split/1`). Deep-malformed inputs reachable only through undocumented shapes (e.g. a bare `-=` arg) were noted as obscure edge behavior, not documented-contract bugs, and are out of scope per the review rules.

