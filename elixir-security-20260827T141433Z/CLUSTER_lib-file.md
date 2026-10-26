# Cluster cross-file findings

#### `CSCAN-cluster-stat-size-undefined-arith`

- **Finding ID** — `CSCAN-cluster-stat-size-undefined-arith`
- **Severity** — Low
- **Confidence** — low (the code-level mismatch and crash mechanism are certain and cited/proven; what I could **not** demonstrate is attacker-reachability of a `:undefined` size on the reviewed Linux platform — see Evidence (c). Treat this as a latent cross-file contract seam, not a confirmed live crash.)
- **Files involved** —
  - `lib/elixir/lib/file/stat.ex:59`, `:66-67`, `:96-98` (contract producer)
  - `lib/elixir/lib/file/stream.ex:134-144` (unguarded consumer)
  - Supporting, out-of-cluster: `kernel/include/file.hrl:28` (authority for the `:undefined`), `lib/elixir/lib/file.ex:489-491` (the `stat → from_record` bridge)
- **Finding shape** — "Safe because the caller checks" broken / trust-boundary contract split: stream.ex relies on an invariant ("`File.stat` `size` is always a non-negative integer") that stat.ex's *own* published contract explicitly does not guarantee.
- **Evidence** —
  - stat.ex extracts the OTP record at `stat.ex:59` (`Record.extract(:file_info, from_lib: "kernel/include/file.hrl")`) and re-exposes it verbatim; its type at `stat.ex:66-67` declares `size: non_neg_integer() | :undefined`. `from_record/1` (`stat.ex:96-98`) copies whatever the record holds — including the atom `:undefined` — straight into `%File.Stat{}` with no coercion. The authority is `file.hrl:28`: `size :: non_neg_integer() | 'undefined'`.
  - `File.stat` routes that struct back to the caller unchanged: `file.ex:489-491` (`{:ok, fileinfo} -> {:ok, File.Stat.from_record(fileinfo)}`).
  - stream.ex consumes it in `count/1`: `stream.ex:135` `:erpc.call(node, File, :stat, [path])`; `stream.ex:136` special-cases only `%{size: 0}`; the next clause `stream.ex:139` `{:ok, %{size: size}} ->` binds `size` with no `is_integer` guard, then `stream.ex:142` does `size = max(size - bom_offset - offset, 0)` followed by `rem/div` at `stream.ex:143-144`. Test (a) confirms that with `size = :undefined` this arithmetic raises `ArithmeticError` ("bad argument in arithmetic expression"), i.e. `Enum.count(File.stream!(path, bytes))` would crash the caller process rather than returning `{:error, …}` or a count.
  - Test (c): across `[]`, `[time: :posix]`, `[raw: true]`, Linux `read_file_info` returned an integer size — I could not reach `:undefined` on this platform (hence the low confidence / Low severity).
- **What the per-file pass missed** — The stat.ex reviewer declared the struct's fields "inert … with no sink in this file" and punted any trust decision to callers, while the stream.ex reviewer treated `size` as an integer from `File.stat` and only analyzed `div/rem`-by-zero via the caller-chosen `bytes` — neither reviewer, reading only their own file, could see that stat.ex's `size :: … | :undefined` contract collides with stream.ex's unguarded arithmetic on that exact value.
- **Suggested fix** — In `stream.ex`, make the byte-count path tolerate a non-integer size the same way it already handles the uncountable case: add/tighten the clause to `{:ok, %{size: size}} when is_integer(size) ->` (or insert `{:ok, %{size: size}} when not is_integer(size) -> {:error, __MODULE__}` before line 139). This confines the value to the module that owns the arithmetic and matches the existing `{:error, __MODULE__}` fallback contract, so `Enum.count` degrades to "uncountable" instead of crashing.
