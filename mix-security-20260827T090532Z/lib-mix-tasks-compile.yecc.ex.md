# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.yecc.ex
<!-- sha256: 5ba565d55f25e006ce1661b0b400f542ea7ca6aaffbb0d49a27edcb5b4301207 -->

_Analysed on 2026-08-27T10:42:14Z_

I've completed a thorough trace of every sink in the file and verified the key behavioral claims by running against the built-from-source toolchain. Here is my report.

## Summary

`Mix.Tasks.Compile.Yecc` is a thin, well-behaved compiler wrapper whose only externally-shaped inputs are boolean CLI switches and `mix.exs` project config (which is RCE-equivalent by Mix's design), and it introduces no sink in this file that crosses a trust boundary — no findings.

## Security

No findings.

### Ruled out

- **Option injection via `:yecc_options` redirecting `parserfile` (line 67, `run/1`)** — The options list is `yecc_options ++ @forced_opts ++ [parserfile: output]`, so a project/dependency-supplied `parserfile` in `:yecc_options` precedes the forced one. I verified yecc resolves duplicate options *last-wins*: with a user `parserfile: "/tmp/yecctest/EVIL.erl"` before the forced `parserfile: ".../g.erl"`, yecc wrote `g.erl` (`Result: {ok,"/tmp/yecctest/g.erl",[]}`); reversing the order wrote `EVIL.erl`. So the forced `parserfile`/`report`/`return` always take effect, matching the moduledoc. Even were it overridable, `:yecc_options` comes only from `mix.exs`, which already executes arbitrary code at build time — no privilege escalation.
- **Atom exhaustion via CLI `args` (line 50, `OptionParser.parse`)** — Non-strict `switches:` mode. I verified `OptionParser.parse` does **not** intern atoms for unknown `--switches`: a runtime-generated flag left `atom_count` delta at `0` and did not become an existing atom (`runtime switch became an atom? false`); unknown switches are dropped. `args` is also supplied by the developer running the build, not a dependency.
- **Code execution via a hostile `.yrl` grammar (line 68, `:yecc.file`)** — The grammar body is *data* parsed by the `:yecc`/`parsetools` library (a different module, out of scope). Grammar action code lands in the generated `.erl` and executes when the parser is later *invoked* at the owning app's runtime, not during `:yecc.file`; the generated `.erl` is written to the source dir and only later compiled by the Erlang compiler (no compile-time execution of that action code). A malicious dependency shipping such a grammar gains nothing over the code it already runs when used.
- **Path traversal in the output `parserfile` (line 67; `output` from `Mix.Compilers.Erlang.compile/6`)** — `output` is `Path.join(dest_dir, module <> ".erl")` where `module = file |> Path.basename() |> Path.rootname()`; `basename`/`rootname` strip any directory and extension, so a discovered filename cannot inject `..` or absolute components. `dest_dir`/source paths come from `:erlc_paths` in `mix.exs`.
- **Manifest deserialization (`binary_to_term` in `clean/0` → `Erlang.clean` → `read_manifest`)** — Lives in `mix/compilers/erlang.ex`, not this file, and is wrapped in `try/rescue`. The manifest is a Mix-written build artifact under `_build/`; reaching it requires local filesystem write access, not a cross-boundary channel exposed here.
- **`Mix.shell().error("…#{project[:app]}…")` (line 76, `preload/1`)** — `#{}` is ordinary Elixir string interpolation (not a `printf`-style format string), and `:app` is the developer's own atom from `mix.exs`, printed to their own terminal. No injection sink.
- **`Mix.ensure_application!(:parsetools)` / `Application.ensure_all_started(:parsetools)` (lines 80–81)** — Literal atoms; no external influence.

## Risks / caveats

- I reviewed only this file. The actual file discovery/read/write and manifest (de)serialization occur in `Mix.Compilers.Erlang` (`mix/compilers/erlang.ex`), and grammar parsing/codegen occur in OTP `:yecc`/`parsetools`; bugs *inside* those (e.g. a yecc parser DoS on a hostile grammar, or `extract_files` following symlinks) would belong to those modules' reviews, not this one. I read `erlang.ex` only to confirm how `input`/`output`/options reach this file's sink.
- The last-wins option precedence and the OptionParser atom behavior were verified empirically on Elixir 1.21.0-dev / OTP 29 built from the source under review; both are behavioral properties of the current tree and would need re-checking if those upstream libraries change.
- The security posture rests on Mix's documented trust model: `mix.exs` (including a dependency's) executes arbitrary code at build time, so config-derived values (`:erlc_paths`, `:yecc_options`) are treated as trusted-equivalent-to-RCE. If that assumption were ever narrowed (e.g. config sourced from an untrusted lockfile/manifest without executing `mix.exs`), the `:yecc_options` pass-through would deserve re-evaluation — though the forced-option ordering already neutralizes the obvious `parserfile` redirect.

