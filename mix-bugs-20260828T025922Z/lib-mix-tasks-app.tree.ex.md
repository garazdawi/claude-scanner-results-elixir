# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/app.tree.ex
<!-- sha256: 75364284d0a2612c0e68ac7f80d6ad18bcc3e5596644f1c9a14e1e74c4f8cb27 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:35:00Z_

## Summary

`Mix.Tasks.App.Tree` (`mix app.tree`) prints an application's dependency tree; the logic largely mirrors upstream and is sound, but the CLI argument dispatch is non-total (crashes with an internal `CaseClauseError` on extra arguments) and the `--output` PNG-hint drops the target directory.

## Bugfixes

#### `CSCAN-app.tree-multi-app-caseclause`
- **Finding ID:** `CSCAN-app.tree-multi-app-caseclause`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `lib/mix/lib/mix/tasks/app.tree.ex:56-69`, `run/1`
- **Class:** Missing clause / non-total function
- **Impact:** Invoking `mix app.tree app_a app_b` (or any invocation with more than one positional argument) raises `CaseClauseError -- no case clause matching: {[], ["app_a", "app_b"]}` instead of a task-level error explaining that at most one application may be given. The user sees an internal Elixir stacktrace that names nothing about the actual mistake, unlike the clean `Mix.raise` used elsewhere in the same function.
- **Evidence:** verified. Running the exact parse-and-match path against `["foo","bar"]`:
  ```
  $ elixir /tmp/t.exs
  {[], ["foo", "bar"]}
  RAISED: CaseClauseError -- no case clause matching:
      {[], ["foo", "bar"]}
  ```
  `OptionParser.parse!/2` returns `{[], ["foo", "bar"]}`, which matches neither `{opts, []}` nor `{opts, [app]}`.
- **Suggested fix:** Add a fallthrough clause, e.g. `{_opts, _many} -> Mix.raise("mix app.tree expects at most one application")`, so misuse produces a helpful message instead of a `CaseClauseError`.

#### `CSCAN-app.tree-png-hint-drops-dir`
- **Finding ID:** `CSCAN-app.tree-png-hint-drops-dir`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `lib/mix/lib/mix/tasks/app.tree.ex:89`, `run/1` (dot branch)
- **Class:** Documentation mismatch (suggested-command inconsistency)
- **Impact:** With `--format dot --output sub/dir/mytree.dot`, the printed "To generate a PNG" hint reads `dot -Tpng "sub/dir/mytree.dot" -o "mytree.png"`. Because `Path.basename/1` strips the directory only from the `-o` target (not the input), following the suggestion writes the PNG into the current working directory rather than alongside the generated `.dot` file. A user who set `--output` to a subdirectory to keep artifacts together gets the PNG somewhere else.
- **Evidence:** verified.
  ```
  $ elixir /tmp/p.exs
  dot -Tpng "sub/dir/mytree.dot" -o "mytree.png"
  ```
- **Suggested fix:** Derive the PNG target from the full path, e.g. `png_file_spec = Path.rootname(file_spec) <> ".png"`, so the suggested output sits next to the input file.

## Risks / caveats

- I could not run the full `mix app.tree` task end-to-end (the sandbox is not inside a Mix project and `bin/elixir` from the build tree was not on the path), so both findings were verified at the level of the exact expressions extracted from `run/1`, not by invoking the task in a real project. The `run/1` logic around them is straightforward, so this does not materially weaken the conclusions.
- `String.to_unsafe_atom/1` on the CLI app name and `--exclude` values (lines 68, 71) grows the atom table from operator-controlled input; this matches the folder's threat-model note but is CLI-only (not network-reachable) and consistent with intended behaviour, so I did not treat it as a defect.
- The `type/1` and `load/2` clause sets are total for every `type` value the callback can produce (`:normal`, `:optional`, `:included`); I confirmed each is covered.

### Ruled out
- **`children_for/1` crashing on `nil` spec** — `Application.spec(app, :applications)` / `:included_applications` return `nil` only for unloaded apps, but `children_for` is reached only after `load/2` returns `true` (app loaded), where those keys default to `[]`; `optional_applications` is explicitly guarded with `|| []`. No `nil -- excluded` crash is reachable.
- **Invalid `--format` value** — an unrecognized format falls through `write`-vs-`print_tree` to `print_tree`, whose `case` has an `_other ->` default; no crash.
- **`type/1` missing clause** — every `type` atom produced by the root (`:normal`) and `children_for` (`:normal`/`:optional`/`:included`) has a matching clause.

