# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/local.ex
<!-- sha256: 72755d46ce26df1dc9d27e62cb07962337cd12fd5558edf1ab16f9ee17687922 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:57:31Z_

## Summary

`mix local` (`Mix.Tasks.Local.run/1`) lists locally-installed archive tasks; the only defect is that tasks lacking a `@shortdoc` are rendered with a literal `# nil` doc column instead of being blank/omitted.

## Bugfixes

#### `CSCAN-local-nil-shortdoc-renders-nil`
- **Finding ID** — `CSCAN-local-nil-shortdoc-renders-nil`
- **Severity** — Low
- **Confidence** — high
- **Location** — `local.ex:19-33` (`run/1`, the `docs`/`Enum.each` block)
- **Class** — Truthiness confusion (unfiltered `nil` from `Mix.Task.shortdoc/1`)
- **Impact** — Any archive task module installed locally that does not define `@shortdoc` is listed as `mix <task>    # nil` rather than with an empty description or being omitted. `Mix.Task.shortdoc/1` is specced `String.t() | nil` (`task.ex:229-235`) and returns `nil` for such modules; the value is threaded straight into the format string `~ts`, and Erlang's `~ts` control renders the atom `nil` as the text `"nil"`. This is inconsistent with `mix help`, whose `build_task_doc_list/1` (`help.ex:293-301`) explicitly filters these out via `if doc = Mix.Task.shortdoc(module)`. Purely a cosmetic/output-correctness defect — no crash.
- **Evidence** — `verified`. `shortdoc/1` returns `nil` when the `:shortdoc` attribute is absent (`task.ex:231-233`). Rendering behaviour reproduced:
  ```
  $ elixir -e ':io_lib.format(~c"mix ~-8s # ~ts", [~c"foo", nil]) |> IO.iodata_to_binary() |> IO.puts'
  mix foo      # nil
  ```
  The `nil` doc reaches `~ts` unfiltered and prints as the literal string `nil`.
- **Suggested fix** — Filter or default the doc, mirroring `help.ex`:
  ```elixir
  docs =
    for module <- modules, doc = Mix.Task.shortdoc(module) do
      {Mix.Task.task_name(module), doc}
    end
  ```
  (or coerce `nil` to `""` before formatting if listing doc-less tasks is intended).

## Risks / caveats

- I could not drive an end-to-end `mix local` run because it requires actually installed archive tasks (`Mix.Local.archives_tasks/0`); the defect is established from the composition of `shortdoc/1`'s `nil` return and the verified `~ts`-on-`nil` rendering, both reproduced directly.
- I considered `run/1` being non-total (only the `[]` clause exists, so `mix local extra-arg` raises `FunctionClauseError`). I did **not** file it: the task documents no arguments and defines a single documented clause, matching upstream; a caller passing extra tokens is outside the documented contract, so it falls under the "contract-violating input is uninteresting" rule. Worth noting only that the resulting error message is unhelpful.
- The `byte_size/1`-based column width (`local.ex:26`) mis-pads task names containing multi-byte UTF-8, but task names are ASCII in practice, so I treated this as non-reachable rather than a finding.

