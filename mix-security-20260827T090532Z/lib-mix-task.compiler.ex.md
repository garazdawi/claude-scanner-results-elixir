# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/task.compiler.ex
<!-- sha256: 25cef2c486b5f1f75844632e729f168bf4bf694a3df6913eb53e9764fd4513df -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/task.compiler.ex
<!-- sha256: 25cef2c486b5f1f75844632e729f168bf4bf694a3df6913eb53e9764fd4513df -->

_Re-analysed on 2026-08-27_

Confirmed: `|>` binds tighter than `==`, so the `maybe_prepend_force/2` predicate parses as `rest == String.replace(to_string(compiler), "_", "-")` — `:my_lang` matches only `--force-my-lang`. No logic bug, and `args` are operator-controlled regardless.

The two first-pass hedges ("would warrant", "did not exhaustively") both concerned `Mix.Sync.PubSub.broadcast/2` and the listener-config machinery. I read `lib/mix/sync/pubsub.ex` and `lib/mix/utils.ex` and resolved both — see **Did I verify this**. Neither becomes a finding: this file is a *publisher* of trusted config-derived data only, and the sole place attacker-crafted bytes are deserialized (`receive_message` → `decode_data`) lives in `pubsub.ex` and is guarded by `Mix.Utils.non_executable_binary_to_term`.

## Summary

`Mix.Task.Compiler` is a thin behaviour-definition and orchestration module. Every code path operates on operator-controlled project config (`Mix.Project.config()`, `config[:compilers]`, `config[:app]`, `config[:build_scm]`) or trusted caller-supplied compiler/args lists. No untrusted (network/archive/git/dependency-data) input reaches any sink located in this file.

## Security

No findings

### Ruled out

- **`Mix.Task.get`/`Mix.Task.run("compile.#{compiler}")` → `Module.concat` atom creation + dynamic dispatch** (lines 246, 261, 299) — `compiler` values come from `compilers/1`, i.e. `config[:compilers] || Mix.compilers()` read from `Mix.Project.config()` (line 218). That is the project's/dependency's `mix.exs` `:compilers` key, executed as code and already full RCE by design (per folder threat model). Downstream resolution reaches `Mix.Utils.command_to_module` → `Module.concat` (unsafe atom-creating concat, `lib/mix/utils.ex:267`), but it is fed only config-derived names, so it is neither an atom-exhaustion nor a module-injection boundary crossing *here*. `manifests/1`/`diagnostics/1` additionally gate dispatch behind `function_exported?/3`. **Confidence: high.**
  - Axes examined: origin of `compiler` (config-derived), presence of an atom/dispatch sink downstream, gating guard. Not examined: none for this file's purview — the sink itself (`Module.concat`) is in `Mix.Utils` and is that file's obligation to review for its *other* callers.

- **`run/2` public API** (`@doc since: 1.19.0`, lines 277–295) — `compilers`/`args` are supplied by trusted callers (e.g. `Mix.Tasks.Compile`) from config and the command line; no documented Mix flow forwards network/archive/dependency *data* as the compilers list. The library caller is not the attacker. **Confidence: high.**

- **`maybe_prepend_force/2` operator precedence** (lines 303–314) — the classic `rest == compiler |> to_string() |> String.replace(...)` gotcha parses in the *safe* direction (`|>` binds tighter than `==`); confirmed by executing it against the elixir built from this source (`--force-my-lang` matches `:my_lang`; `--force-other` does not). Even a misparse would only over-apply `--force` on operator-controlled `args`. **Confidence: high.**
  - Axes examined: precedence (executed), match direction (executed), input trust (`args` operator-supplied). Not examined: none.

- **`notify_modules_compiled/1` → `Mix.Sync.PubSub.broadcast`** (lines 337–362) — `@doc false` (out of scope as public API). Its payload (`config[:app]`, `config[:build_scm]`, the lazy modules-diff, `System.pid()`) is trusted compiler output/config, not attacker data. I read the broadcast implementation (`pubsub.ex:53–125`): the *publisher* lists `port_*` files under a per-user tmp dir, connects to loopback ports, and sends `term_to_binary(message)` outward. This file only ever produces the outbound trusted payload; it never deserializes. The inbound path where a co-resident local process could inject a crafted binary (`pubsub.ex:237–270`) decodes via `Mix.Utils.non_executable_binary_to_term` (`utils.ex:30–34`, which walks terms rejecting funs/executable content) and lives entirely in `pubsub.ex`, not here. **Confidence: high.** *(Upgraded verification of a first-pass hedge — see Did I verify this.)*
  - Axes examined: direction of data flow in this file (publish-only), trust of payload fields, location and safety of the deserialization sink. Not examined: none for this file — the receive-side deserialization is `pubsub.ex`'s obligation and is already covered by the folder-level `non_executable_binary_to_term` invariant.

- **`after_compiler/2`** (line 198) — guarded by `is_atom(name) and is_function(fun, 1)`; registers a caller-trusted callback into `Mix.ProjectStack`. No external input. **Confidence: high.**

- **`__using__/1` macro** (lines 202–212) — compile-time code that executes only when a developer writes `use Mix.Task.Compiler` in their own module; registers a fixed attribute set and sets `@behaviour`. No external input. **Confidence: high.**

- **`normalize/2` `Mix.shell().error(...)`** (lines 328–331) — interpolates `name`/`result` only via `inspect/1`, so no format-string, log, or shell injection; the fall-through is a fail-safe `{:noop, []}`. **Confidence: high.**

- **`reenable/1`** (lines 376–386) — operates on `:all` or a caller/config-derived atom list; interpolates trusted compiler names into `Mix.Task.reenable`. **Confidence: high.**

- **Listener configuration** (moduledoc, lines 34–93) — the `listeners:`/`config :mix, :listeners` mechanism is documented here but *not implemented* here: no code in this file reads `:listeners`, starts a listener, or acts on listener config. The startup machinery is entirely in other modules. There is therefore no sink in this file that listener config can reach, regardless of where the config is populated. **Confidence: high.** *(Resolves the first-pass "would warrant a fresh look" hedge — see Did I verify this.)*
  - Axes examined: whether this file references/consumes `:listeners` (it does not — moduledoc only). Not examined: none for this file.

## Risks / caveats

- The genuinely dangerous sinks (unsafe atom creation via `Module.concat`, dynamic module dispatch, compiler execution, the pub/sub *receive*-side `binary_to_term`) live in **other files** (`Mix.Utils`, `Mix.Task`, `Mix.Sync.PubSub`, the concrete `Mix.Tasks.Compile.*` tasks). This file only forwards trusted names/data into them; those files carry their own review obligation.

## Did I verify this

- **First-pass hedge "did not exhaustively audit `Mix.Sync.PubSub.broadcast`":** RESOLVED. Read `lib/mix/sync/pubsub.ex:53–125` (broadcast/send_message) and `:237–270` (receive_message/decode_data). Confirmed `notify_modules_compiled` is publish-only; the only deserialization of externally-influenceable bytes is on the receive side in `pubsub.ex`, guarded by `Mix.Utils.non_executable_binary_to_term` (read at `lib/mix/utils.ex:30–34`). No untrusted input reaches a sink in `task.compiler.ex`. This is a data-flow verification (which module holds the sink, what the payload contains), appropriately settled by reading — no differential-behaviour axis is involved.
- **First-pass hedge "if listener config could ever be populated from a source other than `mix.exs`/`config :mix`, that would warrant a fresh look":** RESOLVED as moot. `task.compiler.ex` contains no reference to `:listeners` outside the moduledoc and does not start or consume listeners; there is no sink in this file for listener config to reach.
- **`maybe_prepend_force/2` precedence:** verified by execution against the elixir built from this source (unchanged from first pass).
- **`Module.concat`/dispatch reachability:** verified by tracing `compiler` back to `Mix.Project.config()` and reading `Mix.Utils.command_to_module`; the sink is config-fed here. The concat sink's behaviour on *hostile* input remains `Mix.Utils`'s obligation, not clearable from this file.

## Dropped in re-review

None. Every first-pass ruled-out entry is retained; the two hedged caveats are upgraded to verified ruled-out entries (`notify_modules_compiled` payload/broadcast, and listener config), with the neighbour modules read and cited above.
