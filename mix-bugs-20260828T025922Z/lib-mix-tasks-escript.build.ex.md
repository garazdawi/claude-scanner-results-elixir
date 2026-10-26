# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.build.ex
<!-- sha256: c20c28f2745a4ee55780cebb510dd43b3685fc82842984cc06d39b5025792d08 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:54:32Z_

## Summary

`mix escript.build` (`Mix.Tasks.Escript.Build`) compiles a project and packages its BEAM files, config, and a generated `*_escript` entry module into a zip-based escript; after tracing every branch reachable from the sole public entry point `run/1` and verifying the suspicious ones in a live Elixir built from this source, I found no genuine reachable defect.

## Bugfixes

No findings.

## Risks / caveats

- I reviewed only `escript.build.ex`. Behaviour delegated to `Mix.Local.name_for/2`, `Mix.Release.strip_beam/2`, `Mix.Project.*`, `Mix.Dep.cached/0`, and `Config.Reader.*` was treated as a trusted contract and not audited here; a defect in those modules would not surface in this file.
- The generated `gen_main`/`main_body_for` code executes inside the *built* escript at a later time and in a different process; I reasoned about its AST correctness (escaping, unquote depth, merge order) but did not build and run an actual escript end-to-end, so those claims rest on AST analysis rather than a full round-trip.
- Findings that require the caller to supply config violating the documented `@doc` contract (e.g. `config_path: nil`, a non-atom `:main_module`, a non-boolean/non-list `:strip_beams`) were treated as out of scope per the review rules.

### Ruled out

- **`strip_beams/2` never strips because keys are charlists** — the tuple keys are charlists (`~c"...beam"` from `gen_main`, `String.to_charlist/1` from `read_beams`), and `with ".beam" <- Path.extname(basename)` compares against a binary. I worried the match always failed. *Verified* `Path.extname(~c"foo.beam")` returns the binary `".beam"` (its `@spec` is `:: binary`), so the match succeeds and stripping works.
- **`escript_mod`/`emu_args` crash when `:app` is `nil`** — docs allow `app: nil`. `Atom.to_string(nil)` returns `"nil"` (*verified*), so `escript_mod` becomes `:nil_escript` and `start_app_for(nil)` returns `:ok`; no crash.
- **`x in include_priv_for` (a `MapSet`) crashes or mis-answers** — *verified* `:a in MapSet.new([:a])` is `true` and `nil in MapSet.new([:a])` is `false`; `in` dispatches to `Enum.member?/2` correctly for a MapSet.
- **`build_comment/1`, `build_emu_args/2` with `nil` (no `:comment`/`:emu_args` set)** — interpolating `nil` yields the empty string, producing exactly the documented `""`-default output; no crash and behaviourally identical to the documented default.
- **Empty-string `:shebang` replaced by default** — `escript_opts[:shebang] || default`; *verified* `"" || "default"` is `""` (only `nil`/`false` are falsy), so an explicit empty shebang is honoured.
- **Missing deprecation warning for `strip_beam: true`** — the lazy branch only warns on `strip_beam: false`; using the deprecated option with a truthy value gives no warning. This produces the same effective (stripping) behaviour as the default and is a UX nit matching upstream, not a behavioural defect.
- **`main_body_for(:erlang, …)` ignores `runtime_config`** — runtime config is read but discarded for Erlang projects. The `@moduledoc` explicitly documents that `config/runtime.exs` is included only for Elixir escripts, so this is contract-conformant.
- **Double-escaping / wrong unquote depth in `main_body_for(:elixir, …)`** — `compile_config` is `Macro.escape(config)` (or a literal `[]`); it is `unquote`d exactly once in both the runtime and non-runtime branches, and the merge order (`Config.Reader.merge(compile, runtime)`) gives runtime precedence, which is correct.
- **`extra_apps_in_app_tree/1` non-termination / crash** — recursion is bounded by the application dependency DAG, special-cases `:kernel`/`:stdlib`/`:elixir` and the Elixir-family apps, and returns `[]` when `Application.spec/1` is `nil`; no crash on unloadable apps.
- **`parse_strip_beams_options/1` `CaseClauseError`** — only reachable with a `:strip_beams` value that is neither boolean nor list, which violates the documented option contract (out of scope).

