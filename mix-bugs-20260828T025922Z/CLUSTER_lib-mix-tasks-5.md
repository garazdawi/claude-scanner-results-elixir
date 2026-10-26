# Cluster cross-file findings

#### `CSCAN-cluster-tprof-parsehead-divergence`

- **Finding ID** — `CSCAN-cluster-tprof-parsehead-divergence`
- **Severity** — Medium
- **Confidence** — high
- **Files involved** —
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.tprof.ex:191-208` (`run/1` → `parse!/1`)
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.cprof.ex:129`
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.eprof.ex:144`
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/profile.fprof.ex:149`
- **Finding shape** — Inherited assumption broken / contract mismatch across sibling implementations of the same CLI contract.
- **Evidence** — The four `profile.*` tasks implement the *same* documented CLI convention: an option-parse phase feeds the parsed `opts` + `head` into `Mix.Tasks.Run.run(["--no-mix-exs" | args], opts, head, …)`, and everything after the file/expression is forwarded to the executed code as its argv (each moduledoc shows `mix profile.<x> my_script.exs arg1 arg2 arg3`). Three of the four stop option parsing at the first positional argument:
  - `profile.cprof.ex:129` — `OptionParser.parse_head!(args, aliases: @aliases, strict: @switches)`
  - `profile.eprof.ex:144` — `OptionParser.parse_head!(args, …)`
  - `profile.fprof.ex:149` — `OptionParser.parse_head!(args, …)`

  But `profile.tprof.ex:205-207` uses the greedy variant:
  ```elixir
  def parse!(args) do
    {opts, args} = OptionParser.parse!(args, aliases: @aliases, strict: @switches)
    {Enum.map(opts, &parse_opt/1), args}
  end
  ```
  `OptionParser.parse!` (unlike `parse_head!`) parses options at *any* position and raises on an unknown `--flag`, whereas `parse_head!` stops at the first non-switch and leaves the remainder untouched in `head`. Consequently, for `mix profile.tprof my_script.exs --foo bar`:
  - tprof raises `OptionParser.ParseError` on `--foo` (unknown switch), while eprof/cprof/fprof pass `["--foo","bar"]` through to the script (via `Mix.Tasks.Run.run`'s `{file, argv}` split, run.ex:135-143).
  - even a *known* tprof switch placed after the script name (e.g. `my_script.exs --sort calls`) is consumed by tprof instead of being forwarded as a script argument, contradicting the shared "args after the file go to the code" behavior the other three guarantee.
- **What the per-file pass missed** — A single-file reviewer of `profile.tprof.ex` sees a self-consistent, non-crashing `parse!/1` and has no reason to suspect it; only comparing it against its three sibling profilers reveals that tprof breaks the established `parse_head!` positional-argument contract they all share.
- **Suggested fix** — Change `profile.tprof.ex`'s `parse!/1` to use `OptionParser.parse_head!(args, aliases: @aliases, strict: @switches)` (matching cprof/eprof/fprof) so positional/script arguments after the file or expression are forwarded rather than parsed or rejected.
