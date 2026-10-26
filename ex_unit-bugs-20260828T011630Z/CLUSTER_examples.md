# Cluster cross-file findings

No cluster cross-file findings

These two files are independent, standalone example scripts. Each is run on its own (`bin/elixir lib/ex_unit/examples/<file>.exs`), each starts its own ExUnit session with `ExUnit.start(seed: 0)`, and each defines a self-contained module (`Difference` in `difference.exs:8`, `TestOneOfEach` in `one_of_each.exs:8`) with no shared symbols, no cross-referenced functions/macros/structs, and no shared runtime state. There is no signature/arity contract, resource-pairing, state-machine, encode/decode, or inherited-assumption relationship spanning the two files, so no cross-file issue exists to report.
