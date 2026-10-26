# Cluster cross-file findings

#### `CSCAN-cluster-print-app-return-shape`

- **Finding ID** — `CSCAN-cluster-print-app-return-shape`
- **Severity** — Low
- **Confidence** — high
- **Files involved** —
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/process.ex:77-81`
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/io.ex:23-29`
  - `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell/quiet.ex:19`
  - (contract) `/Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/shell.ex:101`
- **Finding shape** — Type / contract mismatch across files (return-shape violation one implementation emits that its siblings and the behaviour contract do not).
- **Evidence** —
  - `shell.ex:101` declares `@callback print_app() :: :ok`.
  - `io.ex:23-29` `print_app/0` runs the `if` for its side effect and then unconditionally `:ok` on line 28 — contract-conforming.
  - `quiet.ex:19` `defdelegate print_app, to: Mix.Shell.IO`, inheriting the `:ok` return — contract-conforming.
  - `process.ex:77-81` `print_app/0` is `if name = ... do send(...) end` with no trailing `:ok`. When there is no printable app name it returns `nil`; when there is one it returns the value of `send/2`, i.e. the message tuple `{:mix_shell, :info, [...]}`. Neither is `:ok`, so the third implementation of the same behaviour disagrees with the other two.
- **What the per-file pass missed** — A single-file reviewer of `process.ex` sees a self-consistent function; only comparing the three `@behaviour Mix.Shell` implementations against the shared `print_app() :: :ok` callback reveals that two return `:ok` and the third does not. The mismatch is currently benign (the sole in-tree caller, `compilers/elixir.ex:796`, discards the result), so severity is Low, but it is a real cross-implementation contract divergence that any future caller relying on the documented `:ok` return would hit only under `Mix.Shell.Process`.
- **Suggested fix** — Make `Mix.Shell.Process.print_app/0` end in `:ok`, mirroring `Mix.Shell.IO`:
  ```elixir
  def print_app do
    if name = Mix.Shell.printable_app_name() do
      send(message_target(), {:mix_shell, :info, ["==> #{name}"]})
    end

    :ok
  end
  ```
