# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/app.ex
<!-- sha256: 43d76c80663853386a4ab7d45bc16ec99beff7347b32bc7f953032499903acaf -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T01:59:35Z_

## Summary
`IEx.App.start/2` is a three-line OTP application callback with no reachable correctness defect.

## Bugfixes

No findings

### Ruled out
- **`with :default <- ... do ... end` without an `else` branch** — When `:shell_multiline_prompt` is already set to a non-`:default` value, the `with` returns that value unmatched. This return value is a discarded statement result (the function's value comes from the subsequent `Supervisor.start_link/2`), so the missing `else` is harmless and the guard correctly means "only install the IEx default prompt when the user hasn't overridden it." Verified by reading: the `with` expression on lines 11-13 is not bound or returned; lines 15-16 produce the function result.
- **Overwriting an explicit `:default` value** — A user who sets `:shell_multiline_prompt` to the atom `:default` gets it replaced by `{IEx.Config, :prompt}`. This is the intended sentinel semantics (`:default` *is* the "unset" marker, matching the `get_env/3` fallback), not a defect.
- **`start/2` return-shape / behaviour-contract violation** — `Supervisor.start_link/2` returns the `{:ok, pid}` / `{:error, _}` shape the `Application` `start/2` callback requires; no demotion or extra element. `@moduledoc false` marks the module internal, but the OTP-invoked callback shape is correct regardless.

## Risks / caveats
- The module is `@moduledoc false`; there is no documented public API surface here beyond the OTP `Application.start/2` callback, which OTP invokes. There is genuinely almost no logic to get wrong, so the absence of findings reflects the file's triviality rather than a shallow review — the entire file is 18 lines and every expression was accounted for above.

