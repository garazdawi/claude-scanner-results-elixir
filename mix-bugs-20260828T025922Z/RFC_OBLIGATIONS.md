<!-- rfc-governed: no -->
<!-- standards: -->

# Documented-contract / behavioural-obligation list — `elixir/lib/mix`

## Scope

This folder is the source tree of **Elixir's `Mix` build tool** (`lib/mix/lib/**` plus its `test/**`). It comprises the `Mix` core, the `mix.tasks.*` task modules, dependency/SCM handling (`Mix.SCM`, `Mix.Dep.*`), config loading, OTP-release/archive/escript packaging, compilers, shells, and inter-process sync primitives. It is application / build-tooling / runtime code written in Elixir (`.ex`/`.exs`), not Erlang.

Nothing in this folder **implements a network protocol, wire format, or cryptographic algorithm** defined by an external protocol/algorithm standard. The candidates a scanner might flag, and why each is `no`:

- **Semantic Versioning** — `Mix.Dep`/`deps.ex` reference SemVer, but version *parsing and matching* is delegated to Elixir's `Version` module (outside this folder); here it only calls `Version.match?` and reports `:nosemver`. No SemVer grammar is implemented here to check.
- **`test.coverage` reports** — built on Erlang's `:cover` module and its native export format; no external LCOV/Cobertura/JUnit XML schema is emitted.
- **OTP releases / escript / archives** (`release.ex`, `escript.build.ex`, `archive.build.ex`) — produce `.rel`/boot-script, escript, and `.ez` (zip) artifacts. These are Erlang/OTP-internal formats, not IETF/W3C/ISO protocol standards.
- **`Mix.SCM.Git`** — shells out to the `git` CLI; it does not implement the git wire protocol.

**Standards references gathered from the application tree:** none were reported, and this scan confirms none govern the folder. There are therefore no external-standard entries to account for.

## Applicable standards / contracts

### External standards
None. The `<!-- standards: -->` marker is intentionally empty; the downstream compliance pass has nothing to adjudicate for this folder.

### OTP-internal contracts (per-file context, NOT compliance obligations)
These are Elixir/OTP conventions the per-file reviewer may find useful. They are **not** external-standard obligations and must not be treated as such:

- **`Mix.Task` behaviour** — task modules `use Mix.Task` and implement `run/1`; `@shortdoc`/`@moduledoc` drive `mix help`.
- **`Mix.SCM` behaviour** (`scm/git.ex`, `scm/path.ex`) — callbacks `fetchable?/0`, `format/1`, `checkout/2`, `update/2`, `checked_out?/1`, `lock_status/2`, `equal?/2`, `managed_deps/0`.
- **`Mix.Shell` behaviour** (`shell/io.ex`, `shell/quiet.ex`, `shell/process.ex`) — `info/1`, `error/1`, `prompt/1`, `yes?/2`, `cmd/2`, `print_app/0`.
- **`Mix.Task.Compiler` behaviour** and `Mix.Task.Compiler.Diagnostic` shapes (`compilers/*.ex`).
- **GenServer / process-registry state holders** — `Mix.State`, `Mix.ProjectStack`, `Mix.TasksServer`, `Mix.Sync.PubSub`, `Mix.PubSub.Subscriber` must tolerate unexpected messages without crashing the build.
- Documented `{:ok, _} | {:error, reason}` and status-tuple return shapes throughout `Mix.Dep.*`, `Mix.Config`, `Mix.Local.Installer`.

## Checkable sub-claims

These are OTP-internal / secure-coding-adjacent obligations offered as *per-file review context only* (they are already within the per-file security review's remit; the RFC-compliance pass does not run here):

1. **DOCUMENTED_RETURN_SHAPE** — Public functions return the tuple shape their `@spec`/`@doc` declares (e.g. `{:ok, _}` / `{:error, reason}`), with no silent demotion to a bare atom. (`@spec`.)
2. **INPUT_INVALID_RETURNS_ERROR** — Parsers of untrusted input (rebar configs, `mix.exs`, lockfiles, `.app` files, config scripts) return an error/status tuple on malformed input rather than raising an uncaught exception. (`@doc` contract.)
3. **TASK_RUN_ARITY_ONE** — Each `Mix.Task` module exports `run/1`; missing/renamed `run` breaks task dispatch. (`Mix.Task` behaviour.)
4. **SCM_CALLBACKS_COMPLETE** — `Mix.SCM` implementations provide every required callback and return the documented `:ok | :error | {:ok, lock}` shapes for `checkout`/`update`/`lock_status`. (`Mix.SCM` behaviour.)
5. **SHELL_CALLBACKS_COMPLETE** — `Mix.Shell` implementations provide `info/error/prompt/yes?/cmd/print_app` with the documented return types. (`Mix.Shell` behaviour.)
6. **HANDLE_INFO_TOLERATES_UNKNOWN** — GenServer state holders (`Mix.State`, `Mix.TasksServer`, `Mix.ProjectStack`, pubsub/sync servers) do not crash on unknown messages; unexpected `handle_info` is logged/ignored. (`GenServer` convention.)
7. **RESOURCE_CLEANED_ON_ERROR_PATH** — Files, ports, temp dirs, ETS tables, and locks acquired by a function are released on every error/exception exit (`try/after`, `File.rm_rf` in cleanup, `Mix.Sync.Lock` release). (`try/after` convention.)
8. **LOCK_RELEASED** — `Mix.Sync.Lock` / build-partition locks acquired are always released even when the guarded work raises, avoiding a stuck build. (Lock-discipline convention.)
9. **EXTERNAL_CMD_NONZERO_HANDLED** — Results of shelling out (`git`, `rebar3`, `System.cmd`) check the exit status and surface a `{:error, _}`/abort rather than proceeding on a failed sub-process. (OTP error-handling convention.)
10. **PATH_TRAVERSAL_GUARDED** — Archive/escript/release unpacking and generator file writes constrain output paths to the intended target directory (no `..` escape from untrusted `.ez`/`.tar`/template input). (Secure-coding; per-file security review.)
11. **ATOM_TABLE_NOT_EXHAUSTED** — Untrusted strings from config/app files are not converted with unbounded `String.to_atom/1`; safe conversion or a bounded set is used. (Secure-coding; per-file security review.)
12. **ERROR_PROPAGATES_NOT_BADMATCHES** — A failed downstream call surfaces as `Mix.raise`/`{:error, reason}`/documented abort, not as a `MatchError`/`CaseClauseError` deep in the stack. (OTP error-handling convention.)
13. **CONFIG_EVAL_SCOPE** — Evaluation of user config/scripts (`Config`, `loadconfig`, rebar `.script`) does not leak or persist unintended global state and restores process/env state afterward. (`@doc` contract.)

## Per-file checking instructions

The RFC-compliance pass does **not** apply to this folder (verdict `no`, empty standards list) — do not raise "RFC/standard non-conformance" findings here. For the per-file review, treat the sub-claims above as OTP-internal / secure-coding context: for each sub-claim that could plausibly apply to the file in hand, look for code that satisfies it. If the file declares the relevant callback, return shape, cleanup (`try/after`), lock release, exit-status check, or path/atom guard, name it. If no satisfying code is present and the file is one where you'd expect the claim to be enforced (a task's `run/1`, an SCM/shell behaviour module, a parser of untrusted input, an unpacker of archives, a lock-holding server), report it as a finding under the appropriate failure-mode class (resource leak, unchecked external command, path traversal, atom exhaustion, missing-callback/return-shape) — not under an RFC/standard-conformance label.