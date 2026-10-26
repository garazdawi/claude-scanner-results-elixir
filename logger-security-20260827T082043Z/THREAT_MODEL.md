# Threat Model — `elixir/lib/logger` (Elixir `Logger` application)

## Protocol / standard implemented
This is Elixir's standard-library **`Logger`** application: a thin, opinionated layer over Erlang/OTP's **`:logger`** (Kernel's structured logging framework, EEP-48-adjacent, documented in the OTP `logger` chapter). It is *not* a network protocol. It provides: log-level filtering, per-process levels, OTP/SASL report translation into Elixir idiom, a default `:logger` formatter (truncation, timezone, ANSI coloring, `$metadata` interpolation), and a deprecated `:gen_event`-based backend shim (`Logger.Backends.*`). No RFC governs it; the governing "spec" is the OTP `:logger` handler/filter/formatter contract and Elixir's own documented `@spec`s.

**Public API entry points:** `Logger` (`log/2-3`, `bare_log/3`, `debug/1`..`error/1`, `metadata/0-1`, `configure/1`, `compare_levels/2`, `put_process_level/2`, `add_backend/2`, `configure_backend/2`); `Logger.Formatter` (`new/1`, `compile/1`, `format/5`, `format_event/2`, `truncate/2`, `prune/1`, `system_time_to_date_time_ms/2`); `Logger.Translator.translate/4`; `Logger.Utils` (`@moduledoc false`, but exported functions are wired as `:logger` primary filters). `Logger.App` is the OTP application callback.

## Trust / contract boundaries
Data reaching this code is almost entirely **in-VM, developer-supplied**, but its *content* is frequently attacker-influenced:

- **Log message payloads & metadata** (`format_event`, `Formatter.format/5`, `Utils.scan_inspect`, `translator`) — the format string and args often embed **untrusted data a remote attacker sent to the application** (usernames, request bodies, headers, decoded protocol frames). The formatter must treat message content as hostile even though the *call site* is trusted. Key sinks: `:io_lib.scan_format`/`build_text`, `Inspect.Algebra`, `truncate_n` (byte arithmetic, UTF-8 fix-up in `fix_binary`), `prune`/`prune_binary`.
- **Format template string** (`Formatter.compile/1`) — operator-controlled config (`config :logger, :default_formatter`). Note `String.to_unsafe_atom` on `$code` fragments — bounded by `@valid_patterns`, but the atom is created before validation.
- **Operator configuration** (`Logger.App.start`, `Handler.default_config`, `configure/1`) — `:level`, `:truncate`, `:sync_threshold`, `:discard_threshold`, `:metadata`, `:translators`, `:backends`. Trusted, but misconfiguration (`:infinity` truncate, huge discard threshold) is a DoS surface.
- **OTP/Erlang log reports** (`Utils.translator`, `Translator.translate/4`) — reports originate from OTP internals *and* from crashing processes whose crash reasons may contain attacker data. Translation runs arbitrary `{mod, fun}` translators and pattern-matches deeply nested report shapes.

Implicit contracts worth watching: `format_event/2` promises chardata out; `truncate/2` is documented **approximate** and "never in the middle of a code point" — the `fix_binary` 13-byte look-back is the load-bearing assumption. `metadata/2` silently drops values that don't match the allow-listed types (documented in moduledoc warning).

## Attacker model
Primary adversary is a **remote/pre-auth peer whose data is logged** by the surrounding application — they don't call `Logger` directly but control string/metadata content. Secondary: a **malicious/careless operator** with config control (in scope only as DoS/footgun, not privilege). Out of scope: co-tenant BEAM processes (Logger is shared infrastructure, not a sandbox boundary) and the local user reading log files (log-file ACLs are the OS's job). The realistic threat classes are **crash/DoS and resource exhaustion inside formatting/translation**, plus **log injection** into downstream sinks.

## Common attack patterns for this code class
- **Log injection / forging** — newline/ANSI/control-char smuggling in message or metadata to spoof log lines or exploit terminal escape sequences. `colorize`/`add_reset` emit raw ANSI; `prune` replaces invalid code points but does **not** strip CR/LF or CSI.
- **Format-string abuse** — `~p`/`~w` inspection of huge/recursive terms; `scan_inspect` truncation pre-pass is the mitigation, verify limits actually bound work.
- **Truncation off-by-one / UTF-8 boundary bugs** — `truncate_n`, `fix_binary`, `prune_binary`: byte math on multibyte code points is the classic bug locus.
- **Unbounded metadata / report size** — `:all` metadata or `:infinity` truncate → memory blowup; deeply nested crash reports in `Translator`.
- **Atom exhaustion** — any `String.to_atom`/`to_unsafe_atom` on externally-shaped input (see `compile/1`).
- **Translator/formatter crash → log-loss cascade** — an exception thrown while formatting a crash report can itself crash the handler.

## Supervisor / process structure
- `Logger.App` — Application callback; installs two **primary `:logger` filters** (`:logger_process_level` → `Utils.process_level/2`, `:logger_translator` → `Utils.translator/2`) and the Elixir default handler. These filters run **in the caller's process** on every log event system-wide — a crash there that isn't rescued affects the logging process, not a supervised child.
- `Logger.Supervisor` (`:one_for_one`) → optionally `Logger.Backends.Internal` (`:rest_for_one`) → starts the `:gen_event` manager named `Logger`, `Logger.Backends.Watcher`, and `Logger.Backends.Supervisor` (`:one_for_one`, `max_restarts: 30 / 3s`).
- `Logger.Backends.Watcher` — `GenServer`, `trap_exit`, wraps each `:gen_event` backend via `add_sup_handler`; converts handler exit into supervised stop. Only present when legacy `:backends` config is set.

## Cross-file invariants
- **`Utils.translator/2` wraps all translation in `try/rescue`** — any exception in `Translator.translate/4` or a third-party translator is caught and converted to a `{:string, chardata}` event (util.ex:34-41). A per-file reviewer flagging an unguarded crash *inside* `Translator` should account for this outer rescue; conversely, translators returning malformed data (not exceptions) bypass it.
- **Truncation is centralized:** `Formatter.format_event → format_message → wrapped_truncate → truncate → Utils.truncate_n`. `wrapped_truncate/2` rescues `ArgumentError` from list truncation (formatter.ex:329-333); `truncate_n/2` *raises* that ArgumentError for non-chardata (utils.ex:239). Reviewers must not judge `truncate_n`'s raise as an uncaught crash in isolation — its list caller rescues it, but the **binary** caller (`wrapped_truncate` on binaries, formatter.ex:327) does **not** rescue, so a raise there would propagate.
- **`scan_inspect` runs a truncation pre-pass over binary args before `:io_lib.scan_format`** (utils.ex:150-165) — downstream inspection width/limit bounds depend on it; don't rule out inspection DoS without checking this ran.
- **Primary filters execute in the logging caller's process**, not inside any supervisor here — so a filter crash is *not* contained by `Logger.Supervisor`; it surfaces wherever `Logger.log` was called.
- **`Logger.Backends.*` is legacy/deprecated** (removal targeted for Elixir v2.0) and only instantiated when the deprecated `:backends` config is present; the default path is the OTP handler `Logger.Backends.Handler` added directly to `:logger`. Threat surface differs between the two paths.