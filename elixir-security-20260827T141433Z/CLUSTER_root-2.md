# Cluster cross-file findings

#### `CSCAN-cluster-debug-info-compile-opts-parse-transform`

- **Finding ID** — `CSCAN-cluster-debug-info-compile-opts-parse-transform`
- **Severity** — Medium
- **Confidence** — high
- **Files involved**
  - `elixir_erl.erl:25-42` — `debug_info(core_v1, …)` (taint source + option concatenation); context: `debug_info(erlang_v1, …)` at `:22-24`, and `dynamic_form/2`'s required `compile_opts` key at `:185`.
  - `elixir_erl_compiler.erl:61-85` — `erl_to_core/2` (`:71-72` executes the option) and `requires_abstract_format/2` (`:78-79` selects the `parse_transform` branch).
- **Finding shape** — Taint across files + "safe because the caller checks" broken (an attacker-controlled, deserialized value flows into a code-execution-capable compiler-option sink in a second file whose own analysis assumed those options were trusted).

- **Evidence**
  - The `debug_info/4` callback is OTP's *public* backend API, invoked by any tool that fetches Core-Erlang debug info from a `.beam` (e.g. analyzers/coverage/decompilers), **without executing that beam's own code**. The stored term `{elixir_v1, Map, Specs}` and its `Map`/`Specs` are fully controlled by whoever crafted the `.beam`.
  - `elixir_erl.erl:27` `#{compile_opts := CompileOpts} = Map,` — `CompileOpts` is taken verbatim from the untrusted debug-info `Map`.
  - `elixir_erl.erl:28` `AllOpts = CompileOpts ++ Opts,` — no filtering. Note `dynamic_form/2` builds a separate sanitized `FilteredOpts` (`elixir_erl.erl:195`) for the emitted `-compile` attribute, but `AllOpts` bypasses that and is used raw.
  - `elixir_erl.erl:32` `elixir_erl_compiler:erl_to_core(Prefix ++ Specs ++ Forms, AllOpts)` — tainted opts (and attacker-controlled `Specs`/forms) cross into the sibling file.
  - `elixir_erl_compiler.erl:78-79` `requires_abstract_format([{parse_transform, _} | _], _Default) -> parse_transform;` — a `{parse_transform, Mod}` at the head of `AllOpts` selects the parse-transform branch.
  - `elixir_erl_compiler.erl:71-72` `compile:noenv_forms(Forms, [… | Opts])` — with `{parse_transform, Mod}` present, `compile` invokes `Mod:parse_transform/2` on the attacker-supplied forms. `erl_to_core`'s own per-file report (`elixir_erl_compiler.erl.md`, "Parse-transform execution … Basis: asserted. Depends on: `elixir_erl.erl` … Opts … carry only `@compile`/env options, not source-string-derived module atoms") explicitly relied on this exact assumption, which the `debug_info` caller violates. The same tainted `AllOpts` is also spread to `compile:noenv_forms(CoreForms, […| AllOpts])` at `elixir_erl.erl:34`.
  - Impact ranges from DoS (feeding hostile `Prefix ++ Specs ++ Forms` to `v3_core:module`, or naming an expensive/crashing transform) up to executing an arbitrary on-system `parse_transform/2` module against attacker-chosen forms during a debug-info fetch. The normal compile path (`load_form/5` → `elixir_erl_compiler:noenv_forms/3`, and `compile/2`) is *not* affected because there `compile_opts` is the module author's own `@compile`/env data — it is only this `debug_info` caller that sources the options from a deserialized term.

- **What the per-file pass missed** — `elixir_erl.erl`'s reviewer dismissed the whole `debug_info` path as "loading an untrusted `.beam` is already arbitrary code" and never traced where `AllOpts` flows, while `elixir_erl_compiler.erl`'s reviewer could see the `parse_transform` execution but had to *assume* (and flagged as an unverified cross-file `Depends on:`) that every caller passes only trusted `@compile`/env options — neither reviewer could see, from one file, that this specific caller feeds attacker-controlled `compile_opts` into that sink.

- **Suggested fix** — Do not pass unfiltered `CompileOpts` from a deserialized debug-info `Map` into the compiler. Strip code-executing options before use, e.g. at `elixir_erl.erl:28` replace with an allowlist/denylist filter such as `SafeOpts = [O || O <- CompileOpts, not is_executable_opt(O)], AllOpts = SafeOpts ++ Opts,` where `is_executable_opt` rejects `parse_transform`/`core_transform` (and any other option that runs a caller-named module). Applying the existing `FilteredOpts` treatment (extended to remove `parse_transform`) to `AllOpts` before lines 32 and 34 is the minimal change.
