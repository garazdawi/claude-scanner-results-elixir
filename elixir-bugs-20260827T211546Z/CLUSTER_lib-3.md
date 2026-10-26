# Cluster cross-file findings

No cluster cross-file findings

I reviewed all eight cluster files (`io.ex`, `json.ex`, `kernel.ex`, `keyword.ex`, `list.ex`, `macro.ex`, `map.ex`, `map_set.ex`) plus their out-of-cluster definitions where relevant, and verified every live cross-file call against its real definition for arity, argument order, return-shape, defaults, and guards. All match:

- `json.ex` → `Macro.struct_info!/2` (macro.ex:1035, returns `[%{field: atom(), ...}]`, read via `& &1.field`), `Macro.generate_arguments/2` (macro.ex:409), `Keyword.get`/`put_new`, `Map.new` — all consistent.
- `map_set.ex:418` → `Map.split_with/2` (map.ex:932): passes the `:sets` v2 internal structure, which is a genuine Erlang map (`is_map/1` holds), and re-wraps both returned submaps as valid `%MapSet{}` — correct, not a leaky-abstraction break.
- `kernel.ex` `get_and_update_in`/`pop_in` machinery → `Map.get_and_update!/3` (map.ex:1052) and `Access.get_and_update`/`Keyword.get_and_update` (keyword.ex:510): the `{get, update} | :pop` contract is produced and consumed symmetrically across all three files.
- `io.ex` → `Keyword.get`/`pop`, `List.to_string`, `IO.iodata_to_binary`; `macro.ex` → `Keyword.fetch`/`take`/`get`, `List.insert_at`/`pop_at`/`to_string`, `Map.fetch` — all arities and guards satisfied by every caller.

Two independent cross-file hunter passes over kernel/keyword/list and macro/io/map corroborated this: no arity, return-shape, default, or guard/precondition mismatch spans any two files in the cluster.
