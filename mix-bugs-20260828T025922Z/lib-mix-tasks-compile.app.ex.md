# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.app.ex
<!-- sha256: 519911e2b2ef306b3351ace0030a495aabb0c6ab123800c163939ad41833ae12 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:40:19Z_

## Summary

`compile.app.ex` generates the `.app` file for a Mix project; it is largely sound, but the `:reliable_dir_mtime` default is wired with `Keyword.get/3` and a function default, so the intended "unreliable on Windows → scan modules" fallback never fires.

## Bugfixes

#### `CSCAN-compile.app-reliable-dir-mtime-eager-default`
- **Finding ID:** `CSCAN-compile.app-reliable-dir-mtime-eager-default`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `compile.app.ex:173`, `run/1` (the `cond` in the `{changed?, modules}` computation)
- **Class:** Truthiness confusion (function default treated as truthy)
- **Impact:** The `:reliable_dir_mtime` option is documented to default to `false` on Windows, whose purpose is to force a directory scan of `.beam` files (branch 3) when the OS directory mtime cannot be trusted. Because the default is supplied to `Keyword.get/3` as a zero-arity function rather than via `Keyword.get_lazy/3`, when the key is absent (the normal case) `Keyword.get` returns the function object itself. A function is truthy, so the `cond` always selects branch 2 (`{false, nil}`) and never scans modules — on every OS, Windows included. On Windows (or any system where the user relies on this option) added/removed `.beam` files are not detected unless some other mtime (config/project/dir) also changed, so the generated `.app` file's `:modules` list can go stale, breaking downstream consumers (releases, tooling) that read it.
- **Evidence:** verified.
  ```
  $ elixir -e 'default = Keyword.get([], :reliable_dir_mtime, fn -> not match?({:win32, _}, :os.type()) end)
    IO.inspect(is_function(default), label: "is_function")
    r = cond do
      false -> :branch1
      default -> :branch2_reliable
      true -> :branch3_scan
    end
    IO.inspect(r, label: "cond result")'
  is_function: true
  cond result: :branch2_reliable
  ```
  The default function is returned unevaluated and is truthy, so the "reliable" branch is taken regardless of `:os.type()`, defeating the Windows fallback entirely.
- **Suggested fix:** Use `Keyword.get_lazy(config, :reliable_dir_mtime, fn -> not match?({:win32, _}, :os.type()) end)` so the default is evaluated to a boolean, or replace the function with the plain expression evaluated eagerly.

## Risks / caveats

- I could only run isolated snippets (the sandbox's latin1 locale warning aside); I did not build a full Windows Mix project to observe the stale-`.app` outcome end to end, but the `cond`-selection behaviour that produces it is verified directly.
- The `String.to_unsafe_atom` use in `modules_from/1` is intentional (module atoms already exist for compiled `.beam` files) and is not an atom-exhaustion bug in scope.
- I reviewed the whole file's `run/1` path and its private helpers; other helpers (`validate_*`, `handle_extra_applications`, `add_compile_env`, `add_modules`) matched their expected shapes and returns under the documented public entry point.

### Ruled out
- **`binary_part(0, byte_size(entry) - 5)` in `modules_from/1`** — guarded by `String.ends_with?(entry, ".beam")`, so `byte_size(entry) >= 5` always holds; no negative-length slice possible.
- **`to_charlist(config[:description] || app)`** — `app` is validated to be an atom before use and `to_charlist/1` handles atoms; no crash.
- **`Map.new(current_properties) != Map.new(properties)` comparison** — order-insensitivity is the intent (keyword lists from consult vs. built list); dedup via `Map.new` is deliberate, not a defect.

