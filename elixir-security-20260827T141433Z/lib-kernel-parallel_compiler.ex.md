# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/parallel_compiler.ex
<!-- sha256: edec4c33b1f03ee5566eb60b2651f3dc11f6ce23784937964dec6bcde4189f9f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:29:35Z_

Both path separators (`/` and `\`) are rejected by module-name validation, so `Path.join(dest, name <> ".beam")` cannot escape `dest`; the other accepted oddities (null, newline, leading-dash) are inert as filenames given `File.write!` takes an explicit path (no shell). This closes the only external-data→filesystem sink. I have enough to write the report.

---

# Security review — `lib/elixir/lib/kernel/parallel_compiler.ex`

## 1. Summary

`Kernel.ParallelCompiler` is compiler-orchestration code whose only untrusted input reachable through its documented public API is the source being compiled — which inherently executes at compile time (the module's documented purpose) — while every other sink resolves to caller-supplied build configuration; no meaningful, separable vulnerability was found.

## 2. Security

`No findings`

The documented public API is `compile/2`, `compile_to_path/3`, `require/2`, `pmap/2`, and the deprecated `async/1`, `files/2`, `files_to_path/3`, `print_warning/1`. I traced every sink in the file (filesystem writes, code loading, path construction, atom handling, process-dictionary state, message dispatch, stderr diagnostics). Each resolves to one of: the inherent code-execution semantics of compilation (the module's documented job), caller-supplied build configuration (`:dest`/`path`/`:beam_timestamp`/callbacks — the caller is not attacking their own build), or the internal compiler message protocol (not part of the public API). The one sink where compiled-artifact data reaches the filesystem is confined by upstream validation, as verified below.

### Ruled out

- **`write_loop` BEAM path built from a module name — `Path.join(path, Atom.to_string(module) <> ".beam")` → `File.write!` (lines 448–449).** This is the only "external-data → filesystem-path" sink. `module` is a module defined by the compiled source, so I checked whether a crafted name (`../…`) could traverse out of `dest`. **Basis: verified.** Compiling `defmodule :"../victim/pwned" do end` is rejected with `ArgumentError: invalid module name` (`:elixir_module.invalid_module_name`) before any write; the traversal-target directory stayed empty. Probing the validator directly: both path separators are rejected (`:"foo/bar"` and `:"a\\b"` → rejected), and the accepted oddities (`:"foo..bar"`, `:"foo bar"`, embedded null/newline, leading `-rf`) contain no separator, so `Path.join` stays inside `dest`, and `File.write!` takes an explicit path (no shell, so a leading `-` is inert). A normal `Foo.Bar.Baz` writes `Elixir.Foo.Bar.Baz.beam` *inside* `out/` — dots are filename characters, not directory separators. Additionally dominated: any route to an arbitrary module atom (`Module.create`, or hand-sending `{:module_available, …}`) requires running code at compile time, which is already full RCE.
- **Compiling/requiring untrusted source executes code (`:elixir_compiler.file`, `:code.load_binary`; lines 660/669/956).** This is the documented behavior of a compiler: macros and module bodies run at compile time. It is not a defect of this module, and there is no "parse/check without executing" mode here that a caller could mistake for a sandbox (tooling that wants that uses `Code.string_to_quoted`, not `ParallelCompiler`). Per the review's severity rules, a finding that reduces to "the victim compiled attacker code" is not a finding about this file.
- **`File.mkdir_p!(path)` / `Code.prepend_path(path)` / `File.touch!(_, timestamp)` / `:filename.join(dest, …)` (lines 398–399, 450, 950).** `path`/`dest`/`timestamp` originate from the caller's `path` argument and `:dest`/`:beam_timestamp` options. A caller choosing its own output directory and timestamp is configuration, not a crossed trust boundary; treating it as attacker input would require the caller to attack itself.
- **`each_cycle` → `{:compile, more, …}` re-queues `more` as files to compile (lines 601–602).** `more` comes from the caller-supplied `:each_cycle` callback; controlling it means controlling the caller. Not an external boundary.
- **`pmap/2` and `async/1` spawn a task per collection item (lines 105–141, 56–60).** The collection is caller-supplied and both functions only work inside an active compilation context (`inner_async` raises `ArgumentError` otherwise). Internal compiler-parallelism helper, no external input.
- **Process-dictionary writes (`:elixir_compiler_info`/`_file`/`_dest`) and the `wait_for_messages` protocol (lines 545–546, 740–899).** These keys and messages are the internal contract between the coordinator and workers it spawns; reaching them requires already executing inside a compilation worker (RCE). Not part of the documented API.
- **No atom-injection, deserialization, command-exec, network, crypto, or archive sinks.** The file contains no `String.to_atom`/`binary_to_atom` on external bytes (only `Atom.to_string`/`Atom.to_charlist`, the safe direction), no `binary_to_term`, no `System.cmd`/`os.cmd`/`System.shell`, no outbound network, no `:crypto`, and no archive extraction. **Basis: read of the full file.**

## 3. Risks / caveats

- **Terminal/log-injection in diagnostics (not reported as a finding).** Error and profile output write file paths and `inspect(module)` to `:stderr` largely unescaped (e.g. `print_error`/`handle_deadlock`/`maybe_log_file_profile`, lines 1047, 1086, 1127). A dependency shipping a source file whose *name* contains ANSI/control bytes could, when that file fails to compile, spoof or corrupt build-log output. I judged this below the reporting bar because the impact is cosmetic build-output integrity, the path values are ultimately the caller's/Mix's file list, and it is a systemic property of compiler diagnostics across the whole codebase rather than a defect in this module's logic — not a module-specific issue to fix here.
- **Robustness, not security:** `write_loop` workers are `spawn_link`ed to the coordinator (line 409), so a `File.write!` failure (e.g. unwritable `dest`) propagates and aborts the whole compile. This is availability/robustness within the caller's own build, not an externally-triggerable security issue, so it is out of scope.
- **Scope boundary:** identifier/Unicode/confusable enforcement (UTS #39, NFC, bidi) lives in the tokenizer and `Module.ParallelChecker`/`String.Tokenizer`, not in this file; module-name *validation* that confines the write path lives in `:elixir_module` (Erlang). I verified the enforcement's effect from this file's perspective by running the built compiler, but the validating code itself is outside the reviewed file and was not audited for bypasses here.
- I did not exercise Windows path semantics; the analysis assumes `/` and `\` (both rejected in module names) are the only path separators, which holds for the BEAM's typical Unix deployment.

