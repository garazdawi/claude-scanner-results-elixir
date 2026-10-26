# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/do.ex
<!-- sha256: 43b94498e9a88eda36ad21247718e18740479fb26d53d1d91cb31eeebddcb265 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:52:43Z_

## Summary

`Mix.Tasks.Do` splits a `mix do` command line into sub-command groups (on `+`, or the deprecated trailing-comma form) and runs each task; the implementation is faithful to upstream Elixir and I found no reachable behavioural defect.

## Bugfixes

No findings

## Risks / caveats

- I reproduced the tokenizer (`gather_commands/1`) logic verbatim and exercised the edge cases (`""`, `","`, trailing-comma, `+`, empty command) with `elixir`; all produced the documented shapes and the empty-command path reaches `Mix.raise` as intended (verified — see `/tmp/t.exs` run output). The bang guard `binary_part(head, byte_size(head), -1) == ","` does **not** crash on an empty-string arg: the `ArgumentError` from `binary_part("", 0, -1)` is raised *inside a guard*, so it is treated as a non-match and the clause is skipped (verified: `gather_commands([""]) => [[""]]`).
- `Mix.Task.run_in_apps/3` and `Mix.Task.reenable/1` both exist in `mix/task.ex`, so the two dispatch paths in `run/1` are valid.
- I did not execute a full `mix do` end-to-end inside a real umbrella project (no project fixture in the sandbox), so `show_forgotten_apps_warning/1`'s umbrella branch is reasoned-about, not run; it uses only standard `Mix.Project` calls and matches upstream.

### Ruled out

- **Atom creation from `--app` values (`String.to_unsafe_atom/1`, line 117).** `Keyword.get_values(:app)` values are mapped through `String.to_unsafe_atom/1`, which creates atoms from external strings and could in principle grow the atom table. Ruled out as a genuine bug: the input is the operator's own command line (not network/peer-controlled), atomization is *required* because app names must be atoms to match project config, and this is the uniform, intentional pattern across all Mix tasks (`deps.ex`, `cmd.ex`, `app.tree.ex`, etc.). `String.to_unsafe_atom/1` is a real exported function in the build under review (verified: `apply(String, :to_unsafe_atom, ["foo"]) => :foo`), so there is no missing-function/compile defect.
- **Deprecated comma splitting a legitimate argument that ends in `,`** (e.g. `mix do run -e "1,"`). The trailing-byte-`,` heuristic will treat such an argument as a separator. Ruled out as a *new* bug: this is the documented, deprecated comma-separator behaviour and is identical to upstream; the `+` form is the recommended replacement.
- **`OptionParser.parse_head!` raising on an unknown option before the first task.** This is the documented bang-variant contract (invalid leading options are genuinely invalid input) and `parse_head!` correctly stops at the first non-switch, leaving task-owned flags like `--list` untouched (verified by inspecting the head-parse semantics; first non-option `"compile"` terminates parsing). Not a defect.

