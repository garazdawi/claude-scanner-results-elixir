# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/task.compiler.ex
<!-- sha256: 25cef2c486b5f1f75844632e729f168bf4bf694a3df6913eb53e9764fd4513df -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:30:26Z_

Confirmed. Bare `:error` — a documented-valid compiler return — falls through to the "malformed return" warning branch and is downgraded to `:noop`. Let me write up the review.

## Summary

`Mix.Task.Compiler` defines the compiler-task behaviour and the driver functions (`compilers/1`, `manifests/1`, `diagnostics/1`, `run/2`, `reenable/1`, `after_compiler/2`). The code is largely clean; the one real defect is that `normalize/2` mishandles a bare `:error` status — a value the documented `@callback run` and moduledoc explicitly permit — silently downgrading a compilation failure to `:noop`.

## Bugfixes

#### `CSCAN-task.compiler-bare-error-downgraded-to-noop`

- **Finding ID:** `CSCAN-task.compiler-bare-error-downgraded-to-noop`
- **Severity:** High
- **Confidence:** medium
- **Location:** `task.compiler.ex:317-335` (`normalize/2`), reached via `run/2` → `run/4` → `run_compiler/2`
- **Class:** Missing clause / non-total function (also a contract violation)
- **Impact:** A compiler whose `run/1` returns the bare atom `:error` — which the `@callback run([binary]) :: status | {status, [Diagnostic.t()]}` (with `status :: :ok | :noop | :error`) and the moduledoc's own bare-atom example (`def run(_args), do: :ok`) both explicitly sanction — is not matched by any of `normalize/2`'s intended clauses. It falls to the catch-all, which prints a spurious `"warning: Mix compiler ... was supposed to return {:ok | :noop | :error, [diagnostic]} but it returned :error"` and returns `{:noop, []}`. Consequences via the documented public `run/2`: (1) the compile failure is reported as success/noop rather than `:error`, so `run/2` returns a non-error status and the build is not failed; (2) because the status is `:noop` rather than `:error`, subsequent compilers in the stack keep running instead of being halted as the `run/2` doc promises ("If a compiler errors, following compilers do not run"); (3) a correct compiler author is told they did it wrong. Bare `:ok`/`:noop` are handled by the second clause but bare `:error` was omitted — an asymmetry inconsistent with the callback contract.

- **Evidence:** `verified`. Reproducing the exact `case` from `normalize/2` standalone:
```
$ elixir -e 'normalize = fn result, name -> case result do
    {status, diagnostics} when status in [:ok, :noop, :error] and is_list(diagnostics) -> {status, diagnostics}
    _ when result in [:ok, :noop] -> {result, []}
    _ -> IO.puts("WARN: compiler #{inspect name} returned #{inspect result}"); {:noop, []}
  end end
  IO.inspect normalize.(:error, :foo)'
WARN: compiler :foo returned :error
{:noop, []}
```
Bare `:error` produces the warning and `{:noop, []}`, whereas `{:error, []}` correctly yields `{:error, []}`.

- **Suggested fix:** Extend the bare-status clause to include `:error`:
```elixir
_ when result in [:ok, :noop, :error] ->
  {result, []}
```

## Risks / caveats

- I could not exercise the full `run/2` path end-to-end (it requires a live Mix project, `Mix.Project` config, and a registered `compile.<name>` task), so the downstream consequences (build not failed, subsequent compilers still running) are reasoned from the surrounding `run/4` control flow rather than run. The `normalize/2` misclassification itself is verified directly.
- Confidence is `medium` rather than `high` because the comment `# ok/noop can come from tasks that have already run` hints the bare-atom clause may have been conceived narrowly for the "already-run" sentinel from `Mix.Task.run`. However, the `@callback` type and the moduledoc's bare-atom `run/1` example make a compiler returning bare `:error` a documented, in-contract usage, so the omission is a genuine mismatch regardless of original intent.
- The `|>` vs `==` precedence in `maybe_prepend_force/2` was checked (verified via AST) and is correct.

### Ruled out

- **`maybe_prepend_force/2` operator precedence** — Verified via `quote` that `rest == compiler |> to_string() |> String.replace("_", "-")` parses as `rest == ((compiler |> to_string()) |> String.replace(...))`; `|>` binds tighter than `==`, matching intent. No bug.
- **`run/4` status accumulation** — Traced all paths: initial `:noop`; any `:ok` promotes to `:ok`; `:error` returns immediately with accumulated diagnostics; `:noop` preserves prior status. Matches the documented "returns `{status, diagnostics}`; if a compiler errors, following compilers do not run." No defect.
- **`compilers/1` with `:xref` inside a piped `if/end`** — The whole `if ... end` is a single expression correctly piped into `maybe_prepend`; `List.delete` removes the (single documented) `:xref` entry; `maybe_prepend` is idempotent against pre-existing `:yecc`/`:leex`. No bug.
- **`reenable/1` / `after_compiler/2` guards raising on non-list / non-atom input** — These require inputs that violate the documented `@spec` (`:all | [atom()]`, `is_atom(name) and is_function(fun,1)`); contract-violating input is out of scope.
- **`notify_modules_compiled/1`** — `@doc false` internal function; not part of the documented public API, and its message shape matches the moduledoc's listener contract regardless.

