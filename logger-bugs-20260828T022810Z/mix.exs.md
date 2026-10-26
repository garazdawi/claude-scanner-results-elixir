# /Users/lukas/code/claude-scanner/elixir/lib/logger/mix.exs
<!-- sha256: 04dee679e4658078338d9191f920edb75a87d24ca0bd4536df7cb8413b45d671 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/mix.exs
<!-- sha256: 04dee679e4658078338d9191f920edb75a87d24ca0bd4536df7cb8413b45d671 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary

`mix.exs` is a static Mix project/application configuration for the `:logger` app with no runtime logic, no public API, and no untrusted-input path, so there is nothing to trigger. On re-review I verified the one cross-file hedge from the first pass (env-key/consumer agreement) by reading every consumer call site; it holds. No findings.

## Bugfixes

No findings.

### Ruled out

- **`truncate: 8096` looks like a typo for `8192`** — This is the long-standing intentional default for Elixir's Logger (bytes at which log messages are truncated); it is a documented configuration value, not a power-of-two size that must be 8192, and no code in this file computes against it. The one consumer that reads it, `Logger.Formatter.new/1` (`formatter.ex:173`, `truncate = options[:truncate] || Application.fetch_env!(:logger, :truncate)`), treats it as an opaque integer limit — any positive integer is valid, so `8096` is not a defect. Not a bug. Confidence: high.
- **`project/0` omits `elixir:` / `deps:` / `build_embedded` etc.** — This is the in-tree bootstrap `mix.exs` for the stdlib `logger` app; the sparse `project/0` (only `app`, `version: System.version()`, `build_per_environment: false`) is the deliberate shape used by the other umbrella-lib projects in the Elixir source tree. `application/0` returns a well-formed keyword list (`registered`, `mod`, `env`) matching the `Mix.Project`/`Application` callback contract. Not a bug. Confidence: high.
- **Any missing-clause / pattern / arithmetic / process defect** — There is no function in this file beyond the two declarative `project/0` and `application/0` accessors that return literal keyword lists; there are no branches, recursion, message handling, or resource acquisition to fail. Searched the entire 38-line file. Not a bug. Confidence: high.

## Cross-file env-key / consumer agreement (upgraded from hedged assumption in first pass)

The first pass deferred this as "out of scope for this file." I verified it directly by grepping every `Application.fetch_env!/get_env` call site across the folder and comparing against the `env:` list declared at `mix.exs:20-35`.

Keys read with **`fetch_env!/2`** (crash on absence) — every one is declared here:

| Key | Consumer | Declared? |
|---|---|---|
| `:translators` | `logger.ex:983`, `app.ex:14`, `handler_test` | ✓ line 23 |
| `:handle_otp_reports` | `app.ex:12` | ✓ line 24 |
| `:handle_sasl_reports` | `app.ex:13` | ✓ line 25 |
| `:metadata` | `app.ex:59` | ✓ line 34 |
| `:translator_inspect_opts` | `translator.ex:159,252,287`, `formatter.ex:324` | ✓ line 29 |
| `:truncate` | `formatter.ex:173` | ✓ line 22 |
| `:utc_log` | `formatter.ex:175`, `handler.ex:32`, `config.ex:100` | ✓ line 21 |
| `:sync_threshold` | `handler.ex:28` | ✓ line 31 |
| `:discard_threshold` | `handler.ex:29`, `config.ex:26` | ✓ line 32 |
| `:discard_threshold_periodic_check` | `config.ex:27` | ✓ line 33 |
| `:start_options` | `internal.ex:138` | ✓ line 30 |

Keys read with **`get_env` + explicit default** (tolerate absence, so no crash risk even though several are not declared here): `:always_evaluate_messages` (line 26, also read with a `false` default at `logger.ex:1161,1268`), `:compile_time_purge_matching` (line 27, default `[]` at `logger.ex:1208`), `:compile_time_application` (line 28, `logger.ex:1200`), plus `:level`, `:backends`, `:console`, `:default_formatter`, `:default_handler`, `:compile_time_purge_level` which are intentionally *not* declared here and always read with a default.

Result: no `fetch_env!` consumer can hit a missing key given this declaration. The first-pass concern ("a missing/renamed key here surfaces as a crash there") is verified as **not present** — the declaration and the crash-on-missing consumers are in agreement. This is a verification, not a finding.

## Did I verify this

- **Env-key agreement**: Verified by reading all consumer call sites (grep across `elixir/lib/logger`), not assumed. Every `fetch_env!` key is declared in `env:`. The neighbour modules named in the standing context (`Logger.Formatter`, `Logger.Backends.Handler`, `Logger.Backends.Config`, `Logger.App`, `Logger.Translator`, `Logger.Backends.Internal`) were the actual consumers checked.
- **Note on scope of this verification**: I confirmed *key presence/spelling* — i.e. that no `fetch_env!` crashes for a missing key. I did **not** re-audit whether each consumer's handling of the declared *value* (e.g. the byte-walk in `Logger.Utils.truncate_n/2` given `truncate: 8096`, or the counter reconciliation in `Config` given `discard_threshold: 500`) is itself correct; that is the consumer files' own review, not a property of this declaration. This is a boundary statement, not a hedge: the value-consumption logic lives in other files and is out of this file's reach.
- **File body**: The two accessors return literal keyword lists with no evaluated logic, so there is no input that reaches executable code. Confidence that this file is defect-free: high.

## Dropped in re-review

None. All prior ruled-out entries are preserved (with confidence fields added), and the sole prior hedge ("whether each consumer tolerates its configured default is out of scope") has been upgraded to the verified cross-file section above rather than dropped.
