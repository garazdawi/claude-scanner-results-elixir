# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/behaviour.ex
<!-- sha256: c3b234325dec8adc2f9b0d14e9d9ac7817a3e4967ce41f1b209cd8b95f16339c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:49:32Z_

Verified: the public `format_diagnostic/1` returns a `:message` binary containing the raw ESC (`0x1B`) and BEL (`0x07`) bytes — the `inspect`-quoted display is only cosmetic; `String.contains?(msg, <<0x1B>>) => true` proves the actual bytes are present. This is exactly what language servers (ElixirLS) and CI/JSON diagnostic consumers ingest.

I have everything I need.

---

# Security review — `lib/elixir/lib/module/behaviour.ex`

## Summary
`Module.Behaviour` is a benign compile-time `@behaviour`/`@impl` checker with no file/network/shell/eval/crypto/ETS sinks; the only real issue is that attacker-controlled callback names from a dependency's `behaviour_info/1` are interpolated raw (un-escaped) into compiler diagnostics, enabling terminal/log/diagnostic control-character injection.

## Security

#### `CSCAN-behaviour-diagnostic-escape-injection`
- **Finding ID:** `CSCAN-behaviour-diagnostic-escape-injection`
- **Severity:** Low
- **Confidence:** high
- **Location:** `behaviour.ex:263-264` (`format_definition/2`), reached via `format_callback/3` (`:110-115`) from the `:missing_callback`/`:callback_mismatch`/`:duplicate_behaviour` warnings; name sourced in `normalize_macro_or_function_callback/1` (`:286`) from `behaviour.behaviour_info(:callbacks)` (`:10`, `:62`); exposed through public `format_diagnostic/1` (`:297`).
- **Class:** Template / interpolation (control-character / terminal & log injection — CWE-150 / CWE-117).
- **Issue:** `format_definition({name, arity})` builds `format_definition(kind) <> " #{name}/#{arity}"`, interpolating the callback/function **name atom raw**. Module names in the very same warnings are passed through `inspect` (which escapes control bytes), but the `name` is not. The `name` originates from the behaviour module's `behaviour_info(:callbacks)` list, which is fully controlled by whoever supplies that behaviour module — i.e. a dependency. An atom callback name may contain arbitrary bytes: ANSI/OSC/BEL escape sequences and newlines.
- **Impact / threat:** A malicious dependency whose behaviour declares a callback named with escape sequences causes those raw bytes to be emitted to the developer's terminal, CI logs, and editor diagnostic panes whenever a project that declares `@behaviour ThatDep` fails to implement the callback (the ordinary, expected trigger for this warning). Consequences: terminal title/clipboard/cursor/screen manipulation (OSC/CSI), hiding or forging *other* compiler warnings (e.g. faking a clean "0 warnings" build), and — via injected newlines — forged log lines that confuse CI log parsing. Trust boundary crossed: **dependency artifact → developer/CI/editor at compile time**. Notably this works even for a *prebuilt/Erlang beam* dependency that is only loaded (`Code.ensure_loaded?`) and never runs code in the victim's build, so it is a genuine (if small) capability gain in that case; and because the raw bytes are embedded in the structured diagnostic `:message`, it also reaches consumers (language servers, JSON diagnostics) that never see a dependency's compile-time stdout.
- **Evidence:** `verified`.
  - Real compiler output (victim only *loads* a precompiled `EvilBeh.beam`; `cat -v` reveals bytes):
    `warning: function pwn^[]0;HACKED-BY-DEP^G^[[31mBOOM/1 required by behaviour EvilBeh is not implemented (in module Victim)` — `^[` = ESC, `^G` = BEL, i.e. `\e]0;…\a` (OSC set-title) + `\e[31m`.
  - Structured message via public API: `Module.Behaviour.format_diagnostic({:missing_callback, Victim, {:"x\e]0;PWNED\a", 1}, :def, EvilBeh})` → `String.contains?(msg, <<0x1B>>) => true`, `<<0x07>> => true`.
- **Suggested fix:** Neutralize the name in `format_definition/2` instead of raw-interpolating — e.g. render the pair via a form that escapes (`Exception.format_mfa`-style, or `" " <> inspect(name) <> "/#{arity}"`), matching the `inspect` already applied to module names in these warnings. Fixing the single `format_definition({name, arity})` clause covers every affected warning path.

### Ruled out
- **`:erlang.list_to_atom(tail)` (`:290`) — atom-table exhaustion.** *Verified.* Input is already an interned atom (`atom_to_list/1` on `function_name` requires it), reached only from a loaded behaviour's `behaviour_info(:callbacks)`; the number of new atoms is ≤ the behaviour's macro-callback count, and loading that behaviour beam already interned every `MACRO-…` atom, so there is no small-input→unbounded-atoms amplification. Repro: compiling `MyBeh` (`@macrocallback my_macro/1`) + impl yields exactly the un-prefixed `:my_macro` with no unbounded growth. The nominal "safe" replacement `list_to_existing_atom/1` is *not* valid here — it would crash legitimate compiles where the un-prefixed macro atom is not yet interned in the compiling process.
- **`Code.ensure_loaded?/1` (`:54`, `:118`, `:184`) — code execution.** Only loads modules the source already names via `@behaviour`/`@impl`; the `@impl Mod` path short-circuits on `behaviour not in behaviours` *before* loading (`:181` precedes `:184`). Beam loading runs no arbitrary code beyond `on_load`; normal compiler behaviour, no escalation.
- **Dynamic dispatch `behaviour.behaviour_info/1` / `module.__protocol__/1` (`:10`, `:61-62`, `:119`) — behaviour dispatch.** Guarded by `function_exported?`/`ensure_loaded?`; `callbacks/1` is only invoked by `Module.make_overridable/2` after it verifies `behaviour_info/1` is exported. Worst case is a `FunctionClauseError`/`UndefinedFunctionError` on the developer's own `defoverridable`, a contract issue, not attacker-reachable.
- **Non-tail recursion in `behaviour_callbacks_for_impls/3` / `callbacks_for_impls/2` (`:201`, `:215`) — stack/CPU DoS.** Driven by `impls = (arity - defaults)..arity`, bounded by function arity (≤255); depth is not source-scalable.
- **`format_diagnostic/1` / `format_warning/1` clause coverage — crash.** An unmatched tuple raises `FunctionClauseError`, but warning tuples are produced internally by this module; supplying an arbitrary tuple is a caller contract violation, not reachable with attacker data.
- No file, network, shell, `Code.eval*`/`compile*`, `binary_to_term`, crypto, ETS/persistent_term, or process-lifecycle sinks are present in this file.

## Risks / caveats
- **Severity framing.** The escape-injection finding is deliberately **Low**: it yields output/log/diagnostic manipulation, not code execution. In the common case where the malicious behaviour is a Hex dependency compiled *from source*, that dependency already has arbitrary compile-time code execution, so the diagnostic path is not an escalation for *that* case — its independent value is the prebuilt/Erlang-beam-only case and the propagation into structured diagnostics (language servers, CI JSON) that a dependency's stdout would not reach.
- **Cross-cutting nature.** Raw `#{name}` interpolation of function/atom names into diagnostics is a general property of the Elixir/Erlang diagnostic subsystem, not unique to this file; I verified and scoped only the instance owned here (`format_definition/2`). A complete remediation belongs at the diagnostic-rendering layer, but a correct, minimal file-local fix exists and is given above.
- **Verification boundary.** I reproduced against the in-tree build (`Elixir 1.21.0-dev`, OTP 29) using `elixirc` with a precompiled behaviour beam; I did not separately exercise `mix compile` or ElixirLS, but both consume the identical diagnostic `:message` binary shown to carry the raw bytes, so the exposure follows directly.
- The `latin1` native-encoding VM warning printed during reproduction is an environment locale artifact and does not affect the findings.

