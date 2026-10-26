# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/utils.ex
<!-- sha256: ff54ec1d321dd91568c877ab0148025916b192c11f32bb341436260c07661f6a -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:43:52Z_

## Summary

`Logger.Utils` — the `:logger` filters (`translator/2`, `process_level/2`, `silence_logger_exit/2`) and the format/truncation helpers (`scan_inspect/4`, `truncate_n/2`) — is correct on all inputs reachable through its documented public contracts; no triggerable in-scope defect was found.

## Bugfixes

No findings.

### Ruled out

- **`translator/2` `else` block raises `TryClauseError` on an out-of-contract translator return (utils.ex:42–54).** A custom translator added via the documented `Logger.add_translator/1` API whose `translate/4` returns anything other than `:none | :skip | {:ok, chardata} | {:ok, chardata, meta}` falls through the `else` clauses and raises `TryClauseError`, which is *not* caught by the `rescue` (the `else` block sits outside the protected region). **Verified** — a `{Bad, :t}` translator returning `:wat` raised `TryClauseError`. Ruled out because it requires the translator callback to violate its documented `c:translate/4` return contract; all four contractually-valid return shapes are handled (`:none`→`:ignore`, `:skip`→`:stop`, `{:ok,_}`, `{:ok,_,_}`). This is misuse of an internal callback contract, not a defect on valid input.
- **`truncate_n/2` splitting a codepoint mid-UTF-8.** Cutting `binary_part(binary, 0, n)` can land mid-codepoint, but `fix_binary/1`+`fix_binary/2` re-validate the trailing (≤13-byte) window, and the second `fix_binary/2` clause deliberately re-buffers orphan continuation bytes so a codepoint straddling the prefix/suffix split is reconstituted. **Verified** — `truncate_n("héllo", 3)` → bytes `[104,195,169]` (`String.valid?` true); sweeping every cut length `n=10..40` over 20×`"é"` and `n=10..60` over 20×`"€"` produced no invalid result. The `@doc`'s "n elements" wording is loose (it truncates by UTF-8 byte width, not element count) but the behaviour is correct and its negative-remainder-on-overrun contract (relied on by `Logger.Formatter.truncate/2`) holds: `remaining < 0` exactly when the input overran.
- **`truncate_n_list/3` improper-tail / boundary clauses.** The `n < 0` clause precedes the list clauses so negative budgets never reach element dispatch; the improper-tail clause builds valid iodata via `:lists.reverse(acc, t)`. **Verified** — `truncate_n(["ab" | "cd"], 3)` → `{["ab" | "c"], -1}`; `truncate_n(["ab", 233, "cd"], 3)` → `{["ab", 233], -1}`.
- **`scan_inspect/4` truncate pre-pass and `~p/~P/~w/~W` rewriting.** The `map_reduce` budget threads correctly (once negative, later binaries collapse to `""` via the `n < 0` clause), and `handle_format_spec` only touches specs whose control char is in `~c"wWpP"`, which always carry a data arg. **Verified** — `scan_inspect(~c"~p", [%{a: 1, b: 2}], 10, %Inspect.Opts{})` produced a well-formed inspected `~s` spec. A `~p` with an empty-args spec (would break `inspect_data/2`) only arises from a format/args-arity mismatch, i.e. caller-supplied bad input, out of scope.
- **`process_level/2` / `silence_logger_exit/2` filter returns.** Both return only legal `:logger` filter verdicts (`:ignore` / `:stop`) and have total catch-all heads. **Verified** — returned `:ignore` on non-matching events.
- **`translator/2` domain-gate ordering (lines 11–14) and `case msg` totality.** The `[:otp | _]`/`[:otp,:sasl|_]`/`[:supervisor_report|_]` clauses correctly stop on the right domain+config combos while passing others through, and the inner `case msg` covers every message shape that survives the earlier `{:string, _}` clause. No unreachable or missing clause found across the ordered heads.

## Risks / caveats

- The `else`-block robustness gap above is genuinely reachable if a third-party translator misbehaves; it is excluded only because it violates the documented callback contract. If the project wants defense-in-depth against buggy external translators, folding the `else` dispatch into the `try` body (or adding a catch-all `else` clause degrading to `{:string, chardata}`) would harden it — but that is a hardening choice, not a contract bug.
- Verification ran against Elixir 1.21.0-dev / OTP 29 built from the tree under review, with `ELIXIR_ERL_OPTIONS="+fnu"` to force UTF-8 name encoding (the VM defaulted to latin1 in this sandbox); the UTF-8 truncation results depend on that encoding being active, which matches production Logger environments.
- `Logger.Utils` is `@moduledoc false`; its functions are nonetheless `@doc`'d and invoked as `:logger` filter/formatter callbacks, so they were treated as the documented public surface for this review.

