# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.cprof.ex
<!-- sha256: 552bcac73ead356a9a5d91480504cfbf5ceef1501627205c7e9890dc78ee0443 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.cprof.ex
<!-- sha256: 552bcac73ead356a9a5d91480504cfbf5ceef1501627205c7e9890dc78ee0443 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`profile.cprof.ex` is the `mix profile.cprof` task wrapping Erlang's `:cprof`; its option-handling and print paths are correct across all documented combinations, but the `:warmup`/`--no-warmup` option is fully implemented yet entirely absent from the documented contract (`@type`, `@spec`, and both doc sections). The previously-hedged Dialyzer consequence has now been reproduced with an executed Dialyzer run and is upgraded to verified.

## Bugfixes

#### `CSCAN-profile-cprof-warmup-undocumented-spec`
- **Finding ID:** `CSCAN-profile-cprof-warmup-undocumented-spec`
- **Severity:** Low
- **Confidence:** high
- **Location:** `profile.cprof.ex:101-105` (`@type profile_opts`), `:184` (`@spec profile/2`), `:31-44` (moduledoc `## Command line options`), `:170-183` (`profile/2` `## Options`)
- **Class:** Contract violation / Documentation mismatch (DOCUMENTED_RETURN_SHAPE / documented-contract obligation)
- **Impact:** The `:warmup` option (default `true`) is read at `:194` and the `--no-warmup` CLI switch is defined at `:120`, but neither is listed in the `profile_opts` type, the `@spec`, the moduledoc command-line-options list, or the `profile/2` `## Options`. A caller who reads the published docs/spec will not know `--no-warmup` exists and cannot disable the (side-effect-producing) warmup run. Because `@spec profile((-> result), profile_opts())` restricts the options to `matching`/`limit`/`module` only, a well-behaved call such as `profile(fun, warmup: false)` — behaviour the code explicitly supports — violates the declared spec, so Dialyzer reports the caller as having no local return. Every sibling profiler (`profile.eprof`, `profile.fprof`, `profile.tprof`) documents `--no-warmup` in the CLI list, `:warmup` in the `profile/2` options, and `warmup: boolean()` in its `profile_opts` type; cprof omits all three. Sibling `profile.eprof.ex` was read this pass and confirms the pattern (`:40` CLI `--no-warmup`, `:112` `warmup: boolean()` in `profile_opts`, `:200` `:warmup` in `## Options`).
- **Evidence:** `verified` (executed).
  - *Runtime honouring* — `elixir /tmp/w.exs` calling `Mix.Tasks.Profile.Cprof.profile(fn -> … end, warmup: false, matching: {:erlang, :_, :_}, limit: 999999)` printed the profile with **no** `Warmup...` line, while the default run in the same script printed `Warmup...`. Confirms the option is honoured despite being undocumented.
  - *Dialyzer consequence — Upgraded from hedged assumption in first pass ("reasoned, not run").* Reproduced with a real Dialyzer run (Erlang/OTP 29, PLT over erts/kernel/stdlib, `ERL_LIBS=/opt/elixir/lib` so `elixir_erl` core extraction works). A module carrying the exact `profile_opts` type and `@spec` from this file: a caller passing only documented options (`matching`, `limit`) produced **no** warning, whereas `profile(fn -> :ok end, warmup: false)` produced `Function caller_warmup/0 has no local return`. The type omission therefore does propagate a Dialyzer error into every caller that legitimately passes `warmup:`. Control (documented-only) clean → warmup-only warned isolates the cause to the missing `warmup` key.
  - *Sibling divergence* — confirmed by reading `profile.eprof.ex` (documents all three) against this file (documents none).
- **Suggested fix:** Add `warmup: boolean()` to the `profile_opts` type (`:101-105`), add `` * `--no-warmup` - skips the warmup step before profiling `` to the moduledoc option list, and add `` * `:warmup` - if the code should be warmed up before profiling (default: `true`) `` to the `profile/2` `## Options`, matching the sibling tasks.

## Risks / caveats

- I exercised the documented public `profile/2` API against the source-built Elixir/OTP-29; the `{limit, module}` dispatch mapped correctly onto `:cprof.analyse/{0,1,2}` and both `print_output/2` clauses matched the correct tuple arity. No crash on any tried combination.
- The two previously-hedged claims ("reasoned, not") are both now resolved by executed runs: the Dialyzer error is reproduced (above), and the runtime honouring of `warmup: false` is directly observed.
- I did not exercise the full CLI path through `Mix.Tasks.Run.run/5` (requires a project/app.start context); the `run/1` wiring is byte-for-byte parity with `profile.eprof.ex` (read this pass, `:143-154` vs cprof `:128-139`), which I treat as correct. This remains an unexercised axis, but it is not the basis of any finding.

### Ruled out

- **`case {limit, module}` non-total / wrong analyse arity** — `Axes examined: limit present/absent × module present/absent (all four combinations run) | Not examined: none`. Ran `{nil,nil}`→`analyse/0`, `{limit,nil}`→`analyse/1`, `{_,module}`→`analyse/1|2`; all returned expected shapes and printed without error.
- **`print_output/2` wrong-shape match** — `Axes examined: 2-tuple (analyse/1) vs 3-tuple (analyse/Mod, analyse/Mod,Limit) return shapes | Not examined: none`. Clauses disambiguated by tuple arity (2 vs 3); `:cprof.analyse/1` returns a 2-tuple and `analyse(Mod)`/`analyse(Mod,Limit)` return 3-tuples (confirmed in `cprof.erl:307-372` and by running both paths); no overlap.
- **`:cprof.start(matching)` rejecting the MFA tuple** — `Axes examined: {m,f,a} MFA tuple with concrete and :_ elements | Not examined: none`. `cprof.erl:90` `start({_,_,_} = MFA)` accepts it; the `matching` run succeeded and reported a reduced match count vs the unfiltered run.
- **`@columns [-60, ...]` negative field width** — `Axes examined: negative width in :io.format applied to string/integer cells (run, output inspected) | Not examined: none`. Erlang `io:format` treats a negative field width as left-justification; observed correctly left-justified module column; intended behaviour.
- **`string_to_existing_module` using `Module.concat` (creates atoms despite the "existing" name)** — `Axes examined: ":"-prefixed (String.to_existing_atom) vs bare-name (Module.concat) module strings for well-formed input | Not examined: pathological/very-large arbitrary module strings as an atom-table-growth vector`. Reachable only via the private CLI `parse_opt` path for `--module`; the resolved atom is passed to `:cprof.analyse` and does not misbehave for well-formed input. Kept as ruled-out for the *wrong-result/crash* class; note the atom-creation-from-CLI-input axis (ATOM_TABLE_NOT_EXHAUSTED) is un-exercised — CLI `--module` strings are operator-controlled, not peer-controlled, so this is not escalated, but it is not fully cleared either.

### Did I verify this?

- **Warmup option honoured at runtime** — YES, executed (`/tmp/w.exs`; `Warmup...` present on default, absent on `warmup: false`).
- **Warmup absent from type/spec/both doc sections** — YES, by reading `:101-105`, `:184`, `:31-44`, `:170-183` of this file.
- **Dialyzer flags callers passing `warmup:`** — YES, executed (was "reasoned, not run" in first pass; now reproduced with a real `dialyzer` run, with a documented-options-only control that stayed clean).
- **Sibling profilers document all three** — YES for `profile.eprof.ex` (read this pass); asserted-by-analogy for fprof/tprof (not re-read this pass, but eprof alone establishes the sibling convention).
- **`{limit, module}` dispatch / print-shape / matching-tuple / negative-width** — YES, executed across the axes listed in Ruled out.
- **Full CLI dispatch through `Mix.Tasks.Run.run/5`** — NO, not executed; relied on byte-for-byte parity with `profile.eprof.ex`. Not load-bearing for any finding.

### Dropped in re-review

None. The single finding is retained and strengthened (Dialyzer sub-claim upgraded from hedged/asserted to verified via executed evidence). No prior finding was dropped.
