# Cluster cross-file findings

No cluster cross-file findings

I read all three cluster files (`mix`, `mix.bat`, `mix.ps1`) plus their out-of-cluster delegation targets (`elixir`, `elixir.bat`) to verify the chains. The three cluster files are mutually consistent:

- `mix` is a bare escript (`Mix.CLI.main()`); it parses nothing and relies on the shebang launcher `elixir` to place the script path + args into the `-extra` program-arg bucket (`elixir:190-197` catch-all). No cross-file contract to violate.
- `mix.bat` delegates faithfully: `call "%~dp0\elixir.bat" "%~dp0\mix" %*` (mix.bat:7). The prepended mix-script path is an unrecognized token in `elixir.bat`'s parser, so it falls straight to `:run` and `-extra %*` re-passes the original argv (elixir.bat:134-145) — behaviourally equivalent to the Unix path.
- `mix.ps1`'s comma-reconstruction loop (mix.ps1:9-27) is designed specifically to reproduce, for the PowerShell entry point, the same space/comma-delimited argv that a cmd.exe caller of `mix.bat` produces. I traced representative array-split cases (e.g. `mix do compile, test`) and the reconstructed `$newArgs` passed to `mix.bat` yields the same effective `%*` string, so there is no parity divergence between the two Windows entry points.

No type/arity mismatch, resource-pairing, state-machine, encode/decode, or inherited-assumption break spans two or more of these files.
