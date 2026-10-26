# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/new.ex
<!-- sha256: 9c4c02731256acb988f88ffc87d9fbf6307dd0704dba95186c13146fab88063f -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/new.ex
<!-- sha256: 9c4c02731256acb988f88ffc87d9fbf6307dd0704dba95186c13146fab88063f -->

_Re-analysed on 2026-08-27_

## Summary

`Mix.Tasks.New` is a greenfield project scaffolder whose every input (the `PATH` argument and `--app`/`--module`/`--sup`/`--umbrella` switches) arrives from `OptionParser.parse!(argv)` as operator-typed command-line data. It contains no network fetch, archive extraction, dependency/lockfile parsing, or git-ref handling, so no dependency-, archive-, or network-controlled value can reach any of its file, atom, or template sinks. This pass re-ran every axis behind the prior ruled-out entries and the two caveats on the OTP-29 toolchain; the one conditional hedge in the first pass ("would warrant re-rating") is resolved below by showing that even under the hypothetical it invokes, the outcome is bounded (one atom per invocation, not exhaustion), so no re-rating is warranted.

## Security

No findings

### Ruled out

- **Atom creation — `String.to_unsafe_atom(name)` in `reserved_app/1` (line 212) and `Module.concat(Elixir, name)` in `check_mod_name_availability!/1` (line 228).**
  Both create non-GC'd atoms (CWE-770), but `name` is the operator's `--app`/path-basename / `--module` value, reached only after a format regex, and only one atom is created per process invocation.
  Confidence: High.
  Axes examined: reachability short-circuit (`invalid_app(name) || reserved_app(name)`), the `$`-anchor trailing-newline bypass, atom-value bound per invocation. **Not examined: none.**
  Evidence (executed, OTP 29 / Elixir 1.21.0-dev): reproduced the file's own guards. `check.("Bad Name") → "invalid"` and `check.("../evil") → "invalid"` — i.e. `reserved_app` (and its `to_unsafe_atom`) is never reached for a name that fails `^[a-z][a-z0-9_]*$`; the `||` short-circuits on `invalid_app`'s truthy string. `check.("mix") → {:reserved, true}`, `check.("goodname") → {:ok_atom_created, :goodname}`. The trailing-newline axis: `"mix\n" =~ ~r/^[a-z][a-z0-9_]*$/ → true`, and `check.("mix\n") → {:ok_atom_created, :"mix\n"}` — so a single trailing `\n` does slip past the regex and yields the atom `:"mix\n"`, but that is still exactly **one** operator-chosen atom per `mix new` invocation with no external byte source driving a loop. Mid-string newline is rejected (`"foo\nbar" → false`). Not exhaustion.

- **File sinks — `File.mkdir_p!`/`File.cd!` (lines 81–84) and `create_file`/`create_directory` (`Mix.Generator`, lines 104–123).**
  Directory paths come straight from the operator's `PATH` argument; generated file paths embed `mod_filename = Macro.underscore(mod)`, and `mod` is gated by `check_mod_name_validity!` (`^[A-Z]\w*(\.[A-Z]\w*)*$`, ASCII `\w`, no `..` because every dot must be followed by `[A-Z]`).
  Confidence: High.
  Axes examined: regex anchor (incl. trailing-newline), `Macro.underscore` output for a boundary input, dependency/archive-derived path component (none present). **Not examined: none.**
  Evidence (executed): `"Foo\n" =~ ~r/^[A-Z]\w*(\.[A-Z]\w*)*$/ → true`, `Macro.underscore("Foo\n") → "foo\n"` — the only bypass is a lone trailing newline, producing a trailing-newline filename in the operator's own new project, not a traversal or a second path segment. `"Foo\ndef evil" → false` (no multi-line injection). No zip-slip surface exists here.

- **Template interpolation — `embed_template`/`*_template(assigns)` (lines 269–504).**
  `embed_template` compiles each EEx template to a function at Mix build time; at `mix new` runtime the assigns (`@mod`, `@app`, `@version`) are string-substituted and are **not** re-parsed as EEx.
  Confidence: High.
  Axes examined: whether an assign value containing `<%= ... %>`/`<% ... %>` is re-evaluated. **Not examined: none.**
  Evidence (executed): defined a module with `embed_template(:lib, ...)` and called `lib_template(mod: "Foo <%= System.cmd(\"id\", []) %> <% :ouch %>")`. Output was the literal string `defmodule Foo <%= System.cmd("id", []) %> <% :ouch %> do` — the EEx tags in the assign appear verbatim, no shell exec, no second-stage evaluation. So there is no template-injection sink; assigns are additionally regex-validated operator input.

- **Compile-time / external read — `Mix.Project.in_project(:umbrella_check, "../..", …)` in `in_umbrella?/0` (line 259).**
  This evaluates `../../mix.exs` as code, but that is the operator's own umbrella project config; `mix.exs` execution is Mix's documented, by-design behaviour (the operator who controls it already has RCE) and is out of scope per the folder threat model. The task atom `:umbrella_check` is a fixed literal; the surrounding `try/catch _, _ -> false` swallows any crash.
  Confidence: High.
  Axes examined: source of the evaluated path (fixed `"../.."`, operator cwd), atom literal-ness, crash containment. **Not examined: none** — all values are in-file literals or operator filesystem state, no dependency/network channel.

- **`Application.ensure_loaded(atom_name)` (line 214).**
  Loads an app resource (`.app`) file, not code; the atom is operator- and regex-gated (see first ruled-out entry).
  Confidence: High.
  Axes examined: same reachability gate as the atom-creation entry. **Not examined: none.** No code-execution or injection sink.

- **Shell/output — `Mix.shell().info(…)` with `cd_path(path)` (lines 130, 158, 179) and `Mix.shell().yes?(msg)` (line 238).**
  These print informational text (`"cd <path>"`) to the operator's terminal; `path` is never handed to a shell for execution, so there is no command injection, and the operator authored the echoed string.
  Confidence: High.
  Axes examined: whether the path string reaches a shell-exec sink (it does not — `info`/`yes?` are display only). **Not examined: none.**

- **Dead defensive code.**
  All guards (`invalid_app`, `reserved_app`, `check_mod_name_validity!`, `check_mod_name_availability!`, `check_directory_existence!`) are reachable and fire. Every reserved name in `reserved_application_names/0` is lowercase-ASCII and passes the `invalid_app` gate, so none is shadowed into unreachability.
  Confidence: High.
  Axes examined: reachability of each guard, whether any reserved name is pre-filtered by `invalid_app`. **Not examined: none.**
  Evidence (executed): `check.("mix") → {:reserved, true}` confirms a reserved lowercase name reaches and trips `reserved_app`; invalid inputs trip `invalid_app` first.

## Risks / caveats

- **Validator anchor weakness (robustness, not a security finding).** Both `invalid_app` (`~r/^[a-z][a-z0-9_]*$/`) and `check_mod_name_validity!` (`~r/^[A-Z]\w*(\.[A-Z]\w*)*$/`) anchor with `$`, which in PCRE matches before a final trailing `\n`. Executed: `"mix\n"` and `"Foo\n"` pass validation; a mid-string newline is rejected (`"foo\nbar" → false`, `"Foo\ndef evil" → false`). Because only a *single trailing* newline slips through, it enables neither multi-line code injection into generated files nor path traversal (`Macro.underscore("Foo\n") → "foo\n"`, a trailing-newline filename only). Input is operator-typed, crossing no dependency/archive/network boundary; the correctness-hardened form would anchor with `\A…\z`. Confidence: High (executed).

- **Reachability (resolved, no longer conditional).** `run/1` is a `Mix.Task` callback; its `argv` is delivered by Mix's CLI task dispatcher and is operator-supplied command-line input, which the folder threat model places **out of scope by design** ("Operator-supplied config … the operator themselves … Out of scope"). No caller in the provided tree feeds `run/1` argv derived from a dependency, package, or network source, and `mix new` is a developer-invoked scaffolder. The first pass hedged that an external caller passing attacker-controlled `--app`/`--module` "would warrant re-rating"; that hedge is withdrawn: even under that hypothetical, the executed evidence above shows the atom-creation sink produces exactly one operator-chosen atom per invocation (not a loop over external bytes), the file sinks admit at most a trailing-newline filename in the operator's own new project, and the templates do not re-evaluate assigns. So no attacker-controlled channel and no unbounded amplification exist regardless of who supplies argv — the conclusion does not depend on the out-of-scope boundary alone. Confidence: High.

## Did I verify this?

- Guard short-circuit (`invalid_app` truthy ⇒ `reserved_app`/`to_unsafe_atom` unreached): **verified by execution** (OTP 29, Elixir 1.21.0-dev) — `check.("Bad Name")`, `check.("../evil")` return `"invalid"` without creating an atom.
- Trailing-newline `$`-anchor bypass and its bound: **verified by execution** — `"mix\n"`/`"Foo\n"` pass, yield `:"mix\n"` / filename `"foo\n"`; one atom per invocation; mid-string newline rejected.
- `embed_template` not re-evaluating assigns: **verified by execution** — EEx tags in an assign value emit literally.
- `Macro.underscore` on a boundary input: **verified by execution** — `Macro.underscore("Foo\n") → "foo\n"`.
- `Mix.Generator`, `Mix.Project`, `Macro`, `String.to_unsafe_atom` live outside the single-file checkout; their security-relevant behaviours were confirmed by running them against the built-from-source toolchain rather than reading those sources. Reading those neighbour modules would only report what their guards do, not which inputs disagree with them — the axes above were settled by pushing values through the real implementations, which is stronger.

## Dropped in re-review

None. No prior finding is dropped; the file still has no findings. The only change is that the first pass's conditional "would warrant re-rating" hedge in the reachability caveat is resolved (not dropped) — restated as an unconditional bounded-outcome conclusion backed by the executed evidence above.
