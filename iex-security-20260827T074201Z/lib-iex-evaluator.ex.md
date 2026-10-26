# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/evaluator.ex
<!-- sha256: b096603152c4d2131f15df93c7063d9db0e57a7b26130ed83a0b271c2a1443dc -->

_Analysed on 2026-08-27T07:53:17Z_

Inventory is confirmed. Let me record the trust-boundary conclusions and write the report.

## Summary
`IEx.Evaluator` is the REPL eval loop whose entire purpose is to compile and run caller-supplied Elixir; every code-execution, file, and parse sink in it operates on input that is trusted-by-design (REPL text and the `.iex.exs` config file), and no path lets network/dependency-derived bytes reach a sink through a documented public API in a way that isn't the intended feature.

## Security

No findings.

### Ruled out

- **`Code.eval_quoted_with_env/3` (lines 259, 383) — arbitrary code execution.** This is the documented purpose of the evaluator. The `expr` at line 383 is the quoted form of what the user typed at the prompt (arrives as `{:eval, ^server, code, counter}`), and line 259 evaluates a fixed `import IEx.Helpers`. Per the folder threat model, evaluator input is trusted-by-design; there is no injection boundary to defend here. Not a defect.
- **`File.read!` + `:elixir.string_to_quoted!` on `.iex.exs` (lines 302–303, `eval_dot_iex`).** The path resolves from `opts[:dot_iex]` / `Config.dot_iex()` or the fixed candidates `./.iex.exs` and `$IEX_HOME|~/.iex.exs` (`load_dot_iex`, lines 280–298). Reading and *evaluating* this file is the documented auto-load feature (verified below), analogous to `.bashrc`/direnv. The filename is fixed (`".iex.exs"`), so no external component drives a path traversal, and the eval is intended. This is a caveat, not a defect in this file — see Risks.
- **Path handling `Path.join/Path.expand` (line 288).** Only the directory prefix varies (`"."`, `$IEX_HOME`, `System.user_home()`); the joined leaf is the constant `".iex.exs"`. No attacker-supplied path component and no confinement to defend. `$IEX_HOME` redirection requires local env control (already code-execution equivalent).
- **`loop/1` message handlers (lines 207–238) — forged control/IO messages.** Every clause pins the sender: `{:eval, ^server, …}`, `{:value_from_binding, ^server, …}`, `{:fields_from_env, ^server, …}`, `{:done, ^server|^ref, …}`. Driving them requires a co-tenant BEAM process that already knows the server/evaluator pids — explicitly out of scope — and even then only touches the user's own REPL session (no cross-tenant asset).
- **`value_from_binding`/`variables_from_binding`/`fields_from_env` (lines 165–205, 218–230) — cross-process readback.** `receiver`/`ref` are caller-chosen and the reply carries binding/env values, but the request is pinned to `^server` and these are the documented public API used by autocomplete. Binding contents are the interactive user's own session state, not a protected secret. No atom creation (`traverse_binding` uses `Map.fetch`; `find_matched_variables` uses `Atom.to_string`, the safe reverse direction).
- **`parse/3` buffer accumulation (lines 74–75, `input = buffer ++ input`).** Repeated `++` on unterminated multi-line input is quadratic, but the input source is the REPL reader feeding user-typed lines (trusted-by-design), not a network/file decoder. Not a security boundary.
- **`case result` parse-error dispatch (lines 113–128).** Only matches 3-tuple error shapes from the tokenizer/parser; a differently-shaped error would raise `CaseClauseError`, but only on trusted REPL text and it is caught by the server loop. No attacker leverage.
- **Terminal-escape / info-leak in `print_error` and `io_inspect` (lines 394–429).** Banners/`Macro.to_string`/stacktrace are written raw to stdio, but their content originates from evaluated (trusted) code or `Macro.Env`/source lines (local trust); `io_inspect` uses `inspect/2`, which escapes control chars in strings. Matches the "content originates from local trust" note in the threat model.
- **Process-dictionary state (`:iex_error`, `:iex_history`, `:iex_whereami`, `:iex_imported_paths`, `:iex_server`, `:iex_evaluator`).** All intra-process scratch state saved/restored within the evaluator process; not cross-tenant shared mutable state.

Verification of the one real trust-boundary behavior (documented, intended):

```
$ cd /tmp/iextest && printf 'IO.puts("PWNED: ...")\n' > .iex.exs && echo ":ok" | iex
PWNED: code from .iex.exs ran as ...
```

Confirms `./.iex.exs` is auto-evaluated on shell start — the intended `load_dot_iex` behavior, not a code defect.

## Risks / caveats

- **`.iex.exs` in the current working directory is auto-evaluated on `iex` / `iex -S mix` startup** (`load_dot_iex` candidate `"."`, verified above). Starting a shell inside a freshly-cloned untrusted repository therefore runs code the repo ships — the direnv/`.envrc`-class risk. This is documented, intended IEx behavior and out of scope as a per-file defect, but it is the one place where "a file a project merely reads" reaches the evaluator; it is worth keeping in mind that the real containment boundary for IEx is *who can reach the group leader / the directory you launch from*, not anything inside this file.
- The genuine attack surface for this module lives elsewhere in the folder (distribution cookie / `remsh`, `IEx.Broker` takeover, and pry consent in `server.ex`/`broker.ex`), consistent with the standing threat model. Within `evaluator.ex` itself the input is trusted-by-design and the exception paths (`safe_eval_and_inspect`, `eval_dot_iex`) are both wrapped in catch-all handlers, so evaluator crashes are contained and do not propagate to the supervised Broker/Pry processes.
- No atom-creation-from-external-bytes, command execution, deserialization, network, archive-extraction, template, or regex-from-input sinks exist in this file; the sink inventory is limited to the code-eval and `.iex.exs`-read paths discussed above.

