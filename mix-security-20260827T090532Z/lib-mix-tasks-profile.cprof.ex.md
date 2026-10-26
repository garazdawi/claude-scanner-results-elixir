# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.cprof.ex
<!-- sha256: 552bcac73ead356a9a5d91480504cfbf5ceef1501627205c7e9890dc78ee0443 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.cprof.ex
<!-- sha256: 552bcac73ead356a9a5d91480504cfbf5ceef1501627205c7e9890dc78ee0443 -->

_Re-analysed on 2026-08-27_

## Summary
`Mix.Tasks.Profile.Cprof` is a developer-invoked profiling wrapper whose entire attack surface is operator-typed CLI argv. Its code-compilation and atom-creation sinks are by-design (this task is `mix run` plus `:cprof` instrumentation) and no untrusted-input boundary is crossed. On re-review I read both neighbour modules the prior report leaned on (`Mix.Tasks.Run` and `Mix.Utils.parse_mfa`) and executed the atom-creation, `parse_mfa`, and `:io.format` behaviours in the sandbox. The one hedged caveat ("…would warrant re-evaluation") is now resolved to a verified trust-boundary statement: nothing in this file or its immediate callers forwards externally-derived strings into the args. No findings.

## Security

No findings

### Ruled out

- **`Code.string_to_quoted!` + `Code.compile_quoted` in `profile_code/2` (lines 148, 155)** — Confidence: High. This is the documented purpose of the task (`mix profile.cprof -e <expr>` / `<script>.exs`). `code_string` is either the `-e` value from argv or `File.read!` of a script path from argv. The operator is out of scope in the folder threat model ("`mix.exs`, dependency build scripts … run arbitrary code … by design"); running the developer's own expression under a profiler is the feature, not a defect. No untrusted (dependency/archive/network/project-file) source reaches this sink.

- **`File.read!(&1)` file-evaluator in `run/1` (line 137)** — Confidence: High (verified by reading the caller). The file-evaluator closure is passed to `Mix.Tasks.Run.run/5`, which I read at `lib/mix/tasks/run.ex:120-174`: the file argument is derived from `head` (CLI argv, lines 135-141) and the evaluator is invoked only inside `if File.regular?(file)` at line 166, with `Mix.raise("No such file: …")` on the else branch. The path is developer-chosen argv; there is no external-source or traversal boundary here (a non-regular/absent path raises rather than reading).

- **`Module.concat([module])` in `string_to_existing_module/1` (line 232)** — Confidence: High (executed). Ran `Module.concat(["Totally.Nonexistent.Xyz1378"])` → returns a freshly-created atom, whereas `String.to_existing_atom(...)` on an unknown name raised `ArgumentError` in the same run. So the non-`:`-prefixed `--module` branch does create atoms (the `":" <> module` branch at line 231 uses the safe `String.to_existing_atom`). Not a finding: `--module` is a single-shot value from developer CLI argv — not sourced from a dependency, archive, git ref, network peer, or project-read file — so there is no atom-exhaustion amplification and no untrusted source. `Module.safe_concat/1` would be strictly safer but the input is operator-trusted.

- **`--matching` → `Mix.Utils.parse_mfa` in `parse_opt/1` (lines 158-165)** — Confidence: High (verified by reading + executed). Read `Mix.Utils.parse_mfa` at `lib/mix/utils.ex:215-250`: it uses the non-`!` `Code.string_to_quoted/1` and returns `:error` on any parse/shape failure. Note `quoted_to_m/2` at line 240 calls `Module.concat(aliases)`, so a well-formed `--matching Foo.Bar` *does* create atoms — but only for the module/function names the developer typed. Executed confirmation: `parse_mfa("Foo.Bar.Baz")` → `{:ok, [Foo.Bar.Baz]}`; `parse_mfa("garbage((")` → `:error` (no crash). The `:error` branch here converts to `Mix.raise("Invalid matching pattern: …")` (line 163). Developer CLI input; no crash, no unbounded/external atom creation.

- **`:io.format` format-string construction in `print_row/2` (lines 282-289)** — Confidence: High (executed). The format string is assembled solely from the module constant `@columns [-60, 12, 5]` and literal format lists (`["s","s","s"]`, `["s","B","s"]`), producing e.g. `~-60s~12B~5s`. Profiling values (module names, counts) are passed only as the `data` arg list, never spliced into the format. Executed confirmation: feeding a data value containing `~s~s~s` printed it literally (no directive interpretation) and `~B` formatted the integer `42` correctly — so there is no format-string injection and no `~s`-on-non-string crash from result data.

- **`profile/2` public API (line 185)** — Confidence: High. Reaching `:cprof.start/analyse` or `apply(fun, [])` with hostile values requires a caller to violate the documented `@spec` (`(-> result), profile_opts()`); the `is_function(fun, 0)` guard rejects non-zero-arity funs. Callers of a public library API are out of scope for the remote/untrusted-content attacker model.

## Risks / caveats
- Verified trust boundary (resolves the prior "would warrant" hedge): I traced `run/1` → `Mix.Tasks.Run.run/5`. The only inputs to the sinks above are `args`/`head`/`opts` produced by `OptionParser.parse_head!` over the argv passed to `run/1`. Neither this file nor `Mix.Tasks.Run.run/5` sources those strings from a dependency, archive, git ref, lockfile, or network peer. This task is not part of `mix deps.get` / `compile` / `archive.install`; it is a manual profiling command whose boundary is the developer's shell. There is therefore no live path by which untrusted content reaches the `Module.concat`/`parse_mfa`/`compile_quoted` sinks — the earlier conditional caveat described a forwarding pattern that does not exist in this file or its immediate callers.

## Did I verify this?
- Read `lib/mix/tasks/run.ex:120-174` — confirmed the file-evaluator is gated by `File.regular?/1` and that `file`/`argv` derive from CLI `head`. **Verified.**
- Read `lib/mix/utils.ex:215-250` — confirmed `parse_mfa` uses non-raising `string_to_quoted`, returns `:error` on failure, and creates atoms via `Module.concat` only from developer-typed names. **Verified.**
- Executed in sandbox: `Module.concat` creates a new atom while `String.to_existing_atom` raises on unknown names; `parse_mfa` returns `{:ok, …}` / `:error` without crashing; `:io.format` does not re-interpret `~` sequences appearing inside data arguments and handles `~B` on an integer. **Verified (ran it).**
- Not fuzzed: I did not exhaustively fuzz `:cprof.analyse/*` return shapes; `print_output/2` handles the two documented shapes (`{count, list}` and `{mod, count, list}`) and `function_text/1` has an `inspect/1` fallback, so a malformed `:cprof` return would raise a `FunctionClauseError` at worst — and `:cprof` is trusted OTP infrastructure, not attacker-controlled. Reasoned statically.

## Dropped in re-review
None. All prior ruled-out entries are retained, and the sole hedged caveat has been upgraded to a verified trust-boundary statement (not dropped, and not converted to a finding, because the executed and read-verified evidence shows no untrusted source reaches the sinks).
