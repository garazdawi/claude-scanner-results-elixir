# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.tprof.ex
<!-- sha256: 5f40fab1f7ddf46c7305b041acbc401db966c72417cd7474b44b0a846837d9f5 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.tprof.ex
<!-- sha256: 5f40fab1f7ddf46c7305b041acbc401db966c72417cd7474b44b0a846837d9f5 -->

_Re-analysed on 2026-08-27 (hedge-resolution pass)_

## Summary
`Mix.Tasks.Profile.Tprof` is a developer-invoked profiling task whose entire attack surface is the developer's own CLI argv and named script files. In this pass I read the two callgraph neighbours the first pass had only assumed about — `Mix.Utils.parse_mfa/1` (`lib/mix/utils.ex:215`) and `Mix.Tasks.Run.run/5` (`lib/mix/tasks/run.ex:120`) — and confirmed both. No sink is reachable from a dependency, archive, git ref, network peer, or project-read file. There are no findings.

## Security

No findings

### Ruled out

- **`Code.string_to_quoted!(code_string)` + `Code.compile_quoted` (`profile_code/2`, lines 210–223)** — `code_string` is either the `-e`/`--eval` expression the developer typed or the contents of the script file they named (the `&profile_code(File.read!(&1), opts)` file-evaluator callback passed to `Mix.Tasks.Run.run/5`). Compiling and running that is the documented purpose of the task (identical to `mix run`), so it crosses no trust boundary; the developer trusts their own code. Confidence: High.
  - Axes examined: origin of `code_string` (eval-expr vs script contents), trust boundary crossed. Not examined: none.

- **`File.read!(&1)` file-evaluator / non-option `head` (`run/1`, lines 195–201)** — **Verified against the neighbour this pass.** `Mix.Tasks.Run.run/5` (`run.ex:120`) computes `{file, argv}` purely from its `head` argument (`run.ex:135–141`), and only invokes `file_evaluator.(file)` after a `File.regular?(file)` check (`run.ex:165–171`); `head` is the developer's non-option argv forwarded verbatim from `parse!/1` here. Any traversal is self-inflicted; no external actor controls the path. Confidence: High.
  - Axes examined: source of `head` (OptionParser non-option args), what `Run.run/5` does with it (derives `file`, gates on `File.regular?`, calls the file callback). Not examined: none.

- **`Mix.Utils.parse_mfa(matching)` → atom creation (`parse_opt/1`, line 226)** — **Verified against the neighbour this pass.** `parse_mfa/1` (`utils.ex:215`) runs `Code.string_to_quoted/1` on the string then `quoted_to_m*`, which mints atoms via `Module.concat(aliases)` (`utils.ex:241`) and passes through bare `is_atom` atoms (`utils.ex:244`). So atoms *are* created, but only from the `--matching` CLI string, which is developer-supplied and consumed once per single-shot CLI invocation. There is no atom-table-exhaustion boundary (CWE-770 not reachable from a hostile source). Confidence: High.
  - Axes examined: does `parse_mfa` create new atoms (yes, via `string_to_quoted` + `Module.concat`), source of the string (`--matching` argv). Not examined: none.

- **`:io.format(@call_format/@measurement_format, to_print)` (`print_row/3`, lines 460–461)** — the format strings are compile-time module-attribute constants; profiling data flows only into the argument list. Re-verified this pass that argument strings never have their `~` directives interpreted: `erl -noshell -eval 'io:format("~-*s ~*s~n", [5, "a~w~p", 3, "b"]), halt().'` printed `a~w~p   b` literally. No format-string injection even if a traced function name contained control sequences. Confidence: High.
  - Axes examined: are the format strings attacker-influenceable (no, compile-time constants), do arg strings get directive-interpreted (no, executed). Not examined: none.

- **`Exception.format_mfa/3`, `inspect/1`, `:erlang.float_to_binary/2`, `Integer.to_string/1` (formatting helpers, lines 417–437)** — operate on real trace-result atoms and numbers for display; no secrets and no downstream interpreter, so no leakage or injection. Confidence: High.
  - Axes examined: presence of a downstream interpreter/sink, secret exposure. Not examined: none.

- **`profile/2` public API (line 277)** — the caller supplies the 0-arity `fun` (guarded `is_function(fun, 0)`) and a typed `profile_opts()` keyword list; opts that violate the `@spec` (e.g. a non-tuple `:matching`) are caller misuse, which is out of scope. Confidence: High.
  - Axes examined: who supplies `fun`/`opts` (the in-process caller), guard on `fun`. Not examined: none.

- **Dead-defensive-code sweep** — the `parse_opt/1` `Mix.raise` catch-alls (invalid `--type`/`--report`/`--sort`), the `get_filter_value!/3` incompatibility guards, `divide(_, 0)`, and `print_output(_type, [])` are all reachable from ordinary CLI input (developer typos or empty result sets); none are unreachable. Confidence: High.
  - Axes examined: reachability of each guard from ordinary CLI input. Not examined: none.

## Risks / caveats
- The behavioural core (`:tprof.profile/2`, `:tprof.inspect/3`) lives in OTP's `:tprof` module, which is not part of this repository and cannot be Read here. Per the folder threat model, OTP library modules are **trusted local infrastructure**, not an attacker-controlled trust boundary — the attacker in this file is developer CLI argv, and `:tprof` is fed only the developer's own `fun` and typed opts. The destructuring `{return_value, {^tprof_type, traces}}` (line 315) would crash the build if `:tprof` returned an undocumented shape, but that is a robustness/availability concern within a single developer invocation, not a cross-boundary vulnerability. This stays a caveat, not a finding, because no untrusted input reaches `:tprof`.
- The eval/require/config/`--no-mix-exs`/`app.start` orchestration is delegated to `Mix.Tasks.Run.run/5`, which I read this pass (`run.ex:120–174`): `file`/`argv` derive from developer `head` and `expr_evaluator` runs on `--eval` exprs. A deeper vulnerability in that shared task would affect every `mix run`-style task equally and is not specific to this file.
- I did not execute a full `mix profile.tprof` end-to-end (it requires a compiled Mix project context); the format-string safety claim was verified directly against `io:format` (re-run this pass), and the remaining conclusions rest on tracing every input to developer-controlled CLI argv/script paths, with the two previously-assumed neighbours now read and confirmed.

## Did I verify this
- **Verified by reading the neighbour (`lib/mix/utils.ex:215–250`):** `parse_mfa/1` creates atoms via `Code.string_to_quoted` + `Module.concat`, only from the `--matching` string. Prior-pass hedge resolved — claim stands.
- **Verified by reading the neighbour (`lib/mix/tasks/run.ex:120–174`):** `Run.run/5` derives the script `file` from developer `head`, gates the file callback on `File.regular?`, and runs `expr_evaluator` on `--eval` exprs. Prior-pass hedge resolved — claim stands.
- **Verified by execution:** `io:format` does not interpret `~` directives inside argument strings (`a~w~p` printed literally).
- **Not verifiable here (stays caveat, not a finding):** the internal return shape of OTP's `:tprof.profile/2` / `:tprof.inspect/3` — module is outside the repo and is trusted local infrastructure fed only developer-controlled input, so it is not an attacker boundary.

## Dropped in re-review
None. The prior pass reported no findings; every ruled-out entry is preserved, and two previously-assumed cross-file claims were upgraded from "assumed" to "verified by reading the neighbour." No ruled-out dismissal was strengthened on an untried axis, and no finding was dropped.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`Mix.Utils.parse_mfa(matching)` → `Module.concat/1` atom creation (`parse_opt/1`, line 226)** — atoms are minted from the `--matching` CLI string only. It is developer-supplied and consumed once per single-shot CLI invocation, so there is no atom-table-exhaustion boundary (CWE-770 not reachable from a hostile source).
