# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/compilers/erlang.ex
<!-- sha256: 7bccaa85365d9857f9a9f551d46e950209f4efbdcb529de1999b235de101b3da -->

_Analysed on 2026-08-27T09:24:37Z_

I now have a complete, verified picture. Let me confirm the reachability summary before writing up:

- **Sink**: `module_from_path/1` (line 374) → `String.to_unsafe_atom/1`, which is exactly `:erlang.binary_to_atom(_, :utf8)` — no existence check, atom never GC'd.
- **Reach**: `compile/6`/`compile_entries/6` → (when `dest_ext == :beam`) `lazy_modules_diff` closure (line 169) → `modules_diff` → `modules_from_paths` → `module_from_path`. Materialized by `Mix.Sync.PubSub.broadcast` **only when a subscriber exists** (verified: `broadcast` returns early on empty subscriber list).
- **Input**: `added`/`changed` derive from source filenames being compiled (dependency-controlled during `mix deps.compile`); `removed` from manifest entries.
- **Verified**: 5043 novel atoms interned; `to_existing_atom` raises on the same input.

The `binary_to_term`/`module.format_error`/`File.rm` concerns all sit behind the `_build`-write boundary (which already implies code execution) and match Mix's universal manifest trust model. Writing the report.

## Summary

The file is Mix's Erlang-family compiler driver; its one genuine (low-severity) issue is that `module_from_path/1` interns a permanent atom per compiled/removed module name via `String.to_unsafe_atom/1`, with the names derived from dependency-controlled source filenames.

## Security

#### `CSCAN-erlang-unsafe-atom-from-filename`
- **Finding ID** — `CSCAN-erlang-unsafe-atom-from-filename`
- **Severity** — Low
- **Confidence** — high
- **Location** — `elixir/lib/mix/lib/mix/compilers/erlang.ex:374`, `module_from_path/1` (reached via `modules_from_paths/1` → `modules_diff/4` → the `lazy_modules_diff` closure at line 169, invoked from `compile_entries/6` when `dest_ext == :beam`).
- **Class** — Atom creation (CWE-770).
- **Impact** — `String.to_unsafe_atom/1` is `:erlang.binary_to_atom(_, :utf8)` with no existence check, and atoms are never garbage-collected. The strings fed to it are the base names of `.beam` outputs in the module diff: `added`/`changed` come from the source-file names being compiled and `removed` from manifest entries. During `mix deps.compile` those source names are controlled by a (malicious or compromised) dependency author, so each uniquely-named `src/<name>.erl` a dependency ships permanently pollutes the compiling VM's atom table. Practical impact is bounded — the count is limited by the number of source files/manifest entries (full exhaustion of the ~1M-atom table would need an absurdly large dependency), the Erlang compiler already interns the module-name atom for each file it compiles, and the diff is only materialized when a `Mix.Sync.PubSub` subscriber exists (`broadcast/2` skips the lazy closure on an empty subscriber list). It is atom-table pollution from cross-boundary input rather than a reliable DoS.
- **Evidence** — `verified`. Reproducing `module_from_path` exactly:
  ```
  $ elixir -e 'm = fn p -> p |> Path.basename() |> Path.rootname() |> String.to_unsafe_atom() end
    b = :erlang.system_info(:atom_count)
    Enum.each(1..5000, & m.("ebin/cscan_evil_module_#{&1}.beam"))
    IO.puts("atoms created: #{:erlang.system_info(:atom_count) - b}")
    try do String.to_existing_atom("cscan_never_seen_x") rescue e -> IO.puts("existing raises: #{inspect e.__struct__}") end'
  atoms created: 5043
  existing raises: ArgumentError
  ```
  `to_unsafe_atom` interns novel atoms unconditionally, whereas `to_existing_atom` raises on the same input — confirming the existence check is bypassed. Definition confirmed at `string.ex:3040` (`def to_unsafe_atom(string) ... :erlang.binary_to_atom(string, :utf8)`).
- **Suggested fix** — Do not convert externally-derived path base names to atoms. The `modules_diff` consumers only need module identity, so keep the names as strings; or, if atoms are required, use `String.to_existing_atom/1` guarded with a fallback (freshly compiled modules already have their atoms interned by the compiler; drop or string-keep `removed` names whose atom does not exist) instead of `String.to_unsafe_atom/1`.

### Ruled out
- **`:erlang.binary_to_term/1` without `:safe` in `read_manifest/1` (line 267).** The manifest lives in `_build`; anyone able to write it already has code execution on the build host (Mix loads `.beam` files from there). This matches Mix's uniform build-local manifest trust model — `compilers/elixir.ex`, `compilers/test.ex`, `compilers/protocol.ex`, `dep/elixir_scm.ex`, and `app_loader.ex` all use plain `binary_to_term`, while only the genuine cross-process boundary (`sync/pubsub.ex`) uses the hardened `Mix.Utils.non_executable_binary_to_term`. Not an escalation across a trust boundary. (See caveats for the amplifier.)
- **Dynamic dispatch `module.format_error(data)` in `to_diagnostics/2` (line 331) and `show_warnings/1` (line 343).** `module`/`data` can originate from the deserialized manifest, so a tampered manifest could dispatch to an arbitrary `format_error/1`. Gated behind the same `_build`-write == code-execution boundary as the item above; no lower-trust source populates warnings.
- **`clean/1` `File.rm/1` on manifest-derived paths (line 204).** Arbitrary file deletion is possible only with a tampered manifest — same `_build`-write boundary.
- **Output-path traversal in `extract_entries/1` (line 250).** `module` = `Path.rootname(Path.basename(file))`, so `/` is stripped and no bare `..` segment can form (verified: `..erl` → filename `..beam`, `...erl` → `...beam`). `Path.join(dest_dir, module <> ".beam")` cannot escape `dest_dir`.
- **`Task.async_stream(..., timeout: :infinity)` (line 153).** A dependency source that hangs the compiler would stall the build, but compiling a dependency already executes its build code — not a distinct security boundary.

## Risks / caveats
- The single reported finding is deliberately rated Low; its exploitability depends on a dependency shipping large numbers of distinctly-named source files and on a `Mix.Sync.PubSub` subscriber being present at broadcast time. I verified the atom-creation primitive and the `broadcast/2` short-circuit directly, but did not stand up a full multi-process partitioned `mix compile` to observe end-to-end atom growth under a live subscriber.
- **Manifest trust amplifier (not counted as a finding):** if you consider a tampered `_build` manifest in scope, `read_manifest/1`'s unsafe `binary_to_term` composes with the `module.format_error(data)` dispatch (arbitrary `format_error/1` invocation) and `clean/1`'s `File.rm` (arbitrary deletion) into a real code-execution/destruction chain. I ruled these out because writing that file already implies code execution on the build host and because the behavior is consistent with every other Mix manifest reader — but if the project's threat model ever treats `_build` manifests as attacker-influenced (e.g. checked-in or artifact-shared build dirs), these three should be revisited together and the manifest read switched to `Mix.Utils.non_executable_binary_to_term`.
- I did not audit the downstream `Mix.Sync.PubSub` subscribers for whether the received module atoms reach `apply/3` or process registration on the *receiving* side; the receive path uses the hardened `non_executable_binary_to_term`, but the atoms interned on the *sending* side (this file) are the object of the finding regardless.

