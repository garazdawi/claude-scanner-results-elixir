# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/release.ex
<!-- sha256: 806609e5f2775c81d539bd8c33e5a0bffc3cc38aeaf100977b6bd5854e9b6c27 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:24:08Z_

## Summary

`Mix.Release` (release assembly for Mix) is largely correct: I exercised every documented public function (`strip_beam/1,2`, `make_sys_config/3`, `make_cookie/2`, `make_start_erl/2`, `make_boot_script/3,4`, `copy_*`) against a source-built Elixir/OTP-29 and all behaved as their `@spec`s promise, with one exception — the `@doc` for `make_boot_script` promises a success shape the function does not return.

## Bugfixes

#### `CSCAN-release-boot-script-ok-shape`
- **Finding ID:** `CSCAN-release-boot-script-ok-shape`
- **Severity:** Low
- **Confidence:** high
- **Location:** `release.ex:598-646`, `make_boot_script/4` (doc at line 604; success return at line 640)
- **Class:** Documentation mismatch
- **Impact:** The `@doc` states the function writes the boot files "returning `{:ok, rel_path}` or `{:error, message}`". The function actually returns the bare atom `:ok` on success (and the `@spec` on line 609 agrees: `:ok | {:error, String.t()}`). A caller who follows the published documentation and writes `{:ok, rel_path} = Mix.Release.make_boot_script(...)` — reasonable given the doc even names a `rel_path` binding it never provides — gets a `MatchError` on every successful call. Callers relying on the `@spec` are unaffected, so the blast radius is limited to doc-driven usage, hence Low.
- **Evidence:** `verified`. Driving the function with a real kernel/stdlib release:
  ```
  $ elixir /tmp/t4.exs
  make_boot_script returns: :ok
  ```
  and the error path:
  ```
  $ elixir /tmp/t5.exs
  error path: {:error, "Unknown application :nope"}
  ```
  Success yields `:ok` (not `{:ok, rel_path}`); the error shape matches the doc. The code (line 640) and `@spec` (line 609) agree on `:ok`, so the `@doc` is the incorrect side of the contract.
- **Suggested fix:** Change the doc sentence from "returning `{:ok, rel_path}` or `{:error, message}`" to "returning `:ok` or `{:error, message}`". (Alternatively, if `{:ok, rel_path}` was intended, change the success clause and `@spec` — but code and spec already agree on `:ok`, so the doc is the defect.)

## Risks / caveats

- `from_config!/3` and all `defp` helpers (`erts_data`, `find_release`, `load_apps`, `merge_mode`, `post_stdlib_applies`, etc.) are `@doc false` / non-exported and only reachable through the `mix release` task with a valid Mix project, so per the scope rules I treated them as out of the documented public API. I noted but did not report: `erts_data/1`'s binary branch (`[_, erts_version] = String.split(basename, "-")`) raises `MatchError` if an `include_erts` path's basename contains more than one hyphen, and `post_stdlib_applies/2` is non-total if a boot script lacks a `stdlib` `start_boot` apply — both require driving `from_config!`/internal state and neither is reachable via the documented functions with well-formed input.
- I verified `strip_beam` success/error/compress shapes, `make_sys_config` valid and invalid-term paths, `make_cookie`, and `make_start_erl` by execution. I did not construct multi-application releases with config providers or `:overlays`/`:tar` steps, so behavior of `copy_erts`/`copy_app` under symlink and cross-platform (`erl` wrapper is Unix-shell only) conditions was reasoned about, not executed; I found no defect there but flag it as unexercised.
- Review rests on the assumption that callers honor the `@spec`s (e.g. `strip_beam`'s `:keep` is `[String.t()]`); contract-violating inputs (a charlist for `:keep`, a non-boolean/non-keyword `:strip_beams`) can raise, but those are out of scope by the stated rules.

### Ruled out
- **`strip_beam/2` return-shape vs `@spec`** — verified success returns `{:ok, binary}` and garbage input returns `{:error, :beam_lib, {:not_a_beam_file, ...}}`, exactly matching `{:ok, binary()} | {:error, :beam_lib, term()}`.
- **`make_sys_config/3` `~s` formatting of the boolean `runtime_config?`** — verified `:io_lib.format("~s", [true])` yields `"true"`; no crash. Invalid-term path returns the documented `{:error, message}`.
- **`make_start_erl/2` charlist interpolation of `erts_version`** — verified `"#{~c"17.0.2"} 0.1.0"` produces `"17.0.2 0.1.0"` (charlist has a `String.Chars` impl); no error.
- **`build_start_clean_boot/1` `for(...) |> Keyword.put` precedence** — the `for` is parenthesized, so the pipe applies to the whole comprehension; `stdlib`/`kernel` are correctly forced to `:permanent`.
- **`validate_steps!/1` tar-after-assemble check** — `Enum.drop_while(steps, &(&1 != :tar))` leaves `[:tar | rest]`; `:assemble in rest` correctly flags an assemble that follows tar and is a no-op when there is no tar.

