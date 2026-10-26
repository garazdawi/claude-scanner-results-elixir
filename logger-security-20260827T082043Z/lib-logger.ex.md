# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger.ex
<!-- sha256: e203c4f4541eb33a91f6dd2610f6b59f42016468cf0c28c6a038f98843a3d2bb -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger.ex
<!-- sha256: e203c4f4541eb33a91f6dd2610f6b59f42016468cf0c28c6a038f98843a3d2bb -->

_Re-analysed on 2026-08-27 (Erlang/OTP 29, Elixir 1.21.0-dev in sandbox)_

## Summary

`logger.ex` is thin public-API glue over Erlang `:logger`. Every content-bearing path forwards the message and metadata **verbatim** to `:logger.macro_log/4` (logger.ex:1051,1058); every dangerous transformation (formatting, truncation, translation, atom/path handling) is performed by other modules. I re-drove the file's forwarding path and confirmed by execution that hostile bytes pass through untouched and that no per-call atom is created. **No findings.**

The prior report's hedges were of two kinds: (1) in-file behavioural claims I had asserted but could now *execute* (verbatim forwarding, no atom creation), and (2) scoping statements pointing at downstream files. Category (1) is now verified by running the code. Category (2) is a genuine file-boundary: the *only* checkable in-file question is whether logger.ex itself transforms content, and it does not — I proved that, so there is no unverified in-file assumption left to upgrade.

## Security

No findings.

Every security-relevant construct was traced to its entry point. The file's job is level/metadata plumbing and compile-time macro expansion; message/metadata *content* — the only genuinely attacker-influenced data per the folder threat model — is passed unchanged to `:logger.macro_log/4`. No atom creation from external strings, no `Code.eval*`, no `binary_to_term`, no file/network/command sink, no path handling beyond the compiler-supplied source path. All entries below are rule-outs.

## Risks / caveats

- The real attack surface named in the folder threat model — log injection (CR/LF/CSI/ANSI smuggling), truncation/UTF-8 boundary bugs, inspection DoS, translator crashes, `$code` atom exhaustion — lives in `formatter.ex`, `utils.ex`, and `translator.ex`. **This is a file boundary, not a hedge:** the only in-file question is whether logger.ex mutates content before forwarding, and I executed the forwarding path (below) to show it does not. Whether the *downstream* handler/formatter sanitizes is a property of those files and is covered by their own reviews; it is not adjudicable here because logger.ex never touches the bytes again after `macro_log`.
- Dangerous parameters of the public functions (handler module in `flush/0`, `{mod,fun}` in `add_translator/1`, config values in `configure/1`) are operator/developer-controlled, not network-forwarded — consistent with the threat model treating operator misconfiguration as an out-of-scope footgun.
- Compile-time paths (`macro_log/4`, `compile_time_purge_matching?/2`, `compile_time_application_and_file/1`) read only trusted root-app `:logger` config and the compiler-provided source path.

### Ruled out

- **Log injection via hostile message/metadata content is *not* introduced or sanitized here (lines 1050–1058, 1062–1066).**
  `verified`. I attached a capturing `:logger` handler and called `Logger.bare_log(:info, "line1\nFAKE [error] injected\e[31mRED\e[0m\r\n", user: "a\nb")`. The handler received `{:string, "line1\nFAKE [error] injected\e[31mRED\e[0m\r\n"}` — every newline, CR, and ANSI/CSI byte intact — and metadata `user: "a\nb"` unchanged, with `domain: [:elixir]` prepended. logger.ex neither strips nor escapes; it forwards verbatim (correct for its role). CR/LF/CSI handling is the formatter's job.
  *Axes examined:* embedded LF, CR, ESC/CSI (`\e[31m`) in message; embedded LF in metadata value — pushed through the live `bare_log → __do_log__ → :logger.macro_log` path and compared the emitted event to the input. *Not examined:* none.

- **`add_elixir_domain/1` crash or corruption on a non-list `:domain` (lines 1062–1066).**
  `verified`. Logging with `domain: :not_a_list` produced `domain: [:elixir]` (the `is_list` guard fails, the fallback clause overwrites — no crash); logging with `domain: [:otp]` produced `domain: [:elixir, :otp]`. No exception, no unbounded growth.
  *Axes examined:* non-list domain value, list domain value — both pushed through the live path. *Not examined:* none.

- **Per-call atom creation in `compare_levels/2` / `elixir_level_to_erlang_level/1` (lines 712–717, 1283–1288).**
  `verified`. After warmup, `Logger.compare_levels(:error, :info)` × 50 000 then × 50 000 produced an atom-table delta of `0` and `0`. (A one-time delta of 144 appears only on the first measurement window and is lazy code/module loading, not per-call — it does not grow with call count: 1 000 calls → 144, 100 000 calls → 144.) No `String.to_atom`/`Module.concat` exists in the file; the level functions only *use* existing atoms.
  *Axes examined:* repeated invocation count (1k / 50k / 100k), warmed vs cold atom-count baseline. *Not examined:* none.

- **`to_string(msg)` fallback in `__do_log__/4` (line 1058) raising on a non-`String.Chars` term.**
  `verified` (crash reproduced) and out of scope (caller-contract). `to_string({:a,:b})`, `to_string(self())`, `to_string(make_ref())` each raise `Protocol.UndefinedError`; `to_string(42)` returns `"42"`. Reaching the `else` branch requires `msg` to be non-binary/non-list/non-map — a value outside the documented `message :: chardata | String.Chars.t() | report()` `@spec`. Remote attacker data arrives as **binaries** (handled by the `is_binary` branch, line 1050), not raw Elixir tuples/pids/refs; producing such a term requires the calling application to violate the spec, and the raise is confined to the caller's own process (it runs before `macro_log` hands off).
  *Axes examined:* term types `{tuple}`, `pid`, `ref` (crash), `integer` (succeeds) — each pushed through `to_string/1`. *Not examined:* none (the only open question, "can an attacker supply a non-binary term," is a caller-contract question, not a divergence; attacker bytes are binaries by construction).

- **`IO.warn` deprecation branch (line 1053) as an amplification/log-noise vector.**
  `verified`. Fires only when the message *type* is non-chardata/non-map (e.g. `Logger.info(42)` — an integer that still stringifies). An attacker cannot set the Elixir *type* of the term; only an application logging a raw decoded scalar as the message hits it, and the effect is a single stderr warning, not a crash or sink. Confirmed the `is_binary/is_list/is_map` gate above governs entry.
  *Axes examined:* integer message (warns, then succeeds), tuple message (warns, then raises in `to_string`). *Not examined:* none.

- **`configure/1`, `configure_backend/2`, `update_translators/1` writing `Application.put_env`/primary-filter config (lines 738–773, 953–963, 981–994).**
  `asserted`. Shared-mutable-state writes, but keys/values are the operator's runtime configuration, not network-forwarded data; `update_translators` serializes via `:elixir_config.serial`. Not reachable from any documented data-carrying API. This is a trust-boundary rule-out, not a two-implementation divergence, so there is no value-carrying axis to push: the input is operator config by definition.
  *Axes examined:* none applicable (trust boundary). *Not examined:* attacker-reachability of operator config — untriable, as reaching these functions is itself the privileged act.

- **`flush/0` dynamic dispatch `module.filesync(id)` (line 789).**
  `asserted`. `module`/`id` come from `:logger.get_handler_config()` (installed handlers), guarded by `function_exported?/3` plus `catch _, _`. The dispatch target is trusted handler config, not external input; installing a handler is an operator act. Trust-boundary rule-out, no data-carrying axis.
  *Axes examined:* none applicable. *Not examined:* attacker ability to install a malicious handler — untriable (privileged act, out of threat model per the co-tenant exclusion).

- **`compare_levels/2` / `elixir_level_to_erlang_level/1` raising on an invalid level atom.**
  `verified` (crash reproduced) and out of scope. `Logger.compare_levels(:bogus, :info)` raises `ArgumentError` from `:logger.compare_levels`; inputs violate the `level()` `@spec`. At compile time the level is a source-literal atom, so a bad level fails the build loudly rather than as a runtime security event. `:warn` is handled (returns `:gt`, with a deprecation warning).
  *Axes examined:* invalid atom `:bogus` (raises), deprecated `:warn` (handled). *Not examined:* none.

- **Compile-time macro expansion (`macro_log/4`, `escape_metadata/1`, `no_log/2`, lines 1132–1281).**
  `asserted`. `unquote`s only the developer's own message/metadata AST (ordinary compilation of source the compiler already compiles) and `Macro.escape`s the literal `mfa`; it never parses external strings into code. `compile_time_application_and_file/1` uses `String.to_charlist` (line 1201), **not** `to_atom`. No unquote-hygiene or compile-time-execution injection. Trust-boundary rule-out (config + developer AST); no network-carried axis exists.
  *Axes examined:* none applicable (compile-time, trusted inputs). *Not examined:* none reachable — external data never enters this path.

- **`bare_log/3` `Map.new(metadata)` / metadata reducers (lines 595–609, 1009).**
  `verified` (behaviour) / `asserted` (cost). O(n) over caller-built metadata with no atom creation or dangerous sink. In the live run above, `Map.new([user: "a\nb"])` forwarded the value unchanged. Cost and content are set by the calling application, not forwarded network bytes.
  *Axes examined:* keyword metadata with hostile string value — pushed through live path. *Not examined:* none.

## Did I verify this?

Yes, by execution, for every in-file behavioural claim:
- Ran a live capturing `:logger` handler and drove `Logger.bare_log/3` with a CR/LF/ESC/CSI-laden message and a newline-bearing metadata value → confirmed **verbatim** forwarding and `domain: [:elixir]` injection.
- Ran non-list vs list `:domain` through the live path → confirmed no crash, fallback-overwrite vs prepend semantics.
- Measured atom-table deltas across 100 000 `compare_levels/2` calls (warm) → `0` per-call atoms.
- Reproduced `to_string/1` `Protocol.UndefinedError` on tuple/pid/ref and success on integer.
- Reproduced `ArgumentError` on an invalid level atom and the handled `:warn` path.

The scoping statements about `formatter.ex`/`utils.ex`/`translator.ex` are file boundaries, not unverified in-file assumptions: I proved logger.ex does not transform content, so downstream sanitization is genuinely another file's property and correctly out of scope here.

### Dropped in re-review

None. The prior report reached "No findings"; every rule-out is preserved, and the two in-file claims that were previously `asserted` on reasoning (verbatim forwarding, zero per-call atom creation) are now `verified` by execution. No prior finding was dropped, and no assumption warranted upgrading to a low-confidence finding — the only in-file behaviours that could carry a defect were executed and behaved correctly.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 2 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`IO.warn` amplification in the same deprecation branch (line 1053)** — same gate: only fires for messages whose *type* is non-chardata/non-map. An attacker cannot set the Elixir type of the term; only an app that logs a raw decoded scalar (`Logger.info(Jason.decode!(body))` where `body` is `42`) as the message itself would hit it, which is app misuse, and the effect is a stderr warning, not a crash or RCE.

- **Log injection via hostile message content** — `verified`: `Logger.bare_log(:info, "line1\nFAKE [error] injected\e[31m…")` emits the embedded newline and raw ANSI/CSI unchanged. This confirms the file passes chardata through verbatim (correct for its role); any CR/LF/CSI stripping is the formatter's responsibility (`prune`/`colorize` in `formatter.ex`), so this is not a logger.ex defect.
