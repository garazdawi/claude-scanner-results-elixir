# Cluster cross-file findings

No cluster cross-file findings

After reading all eight cluster files (`elixir.erl`, `elixir_aliases.erl`, `elixir_bitstring.erl`, `elixir_bootstrap.erl`, `elixir_clauses.erl`, `elixir_code_server.erl`, `elixir_compiler.erl`, `elixir_config.erl`) plus the out-of-cluster definitions the analysis depended on (`elixir.hrl`, `elixir_env.erl`, `elixir_sup.erl`, `elixir_tokenizer.erl` return shapes, `elixir_module:compile/6`), every cross-file contract in this cluster holds:

- **Config `get/1` no-fallback contract (threat-model invariant).** Every `elixir_config:get/1` key used in-cluster (`parser_options`, `tracers` in `elixir_compiler.erl:11,19`) is pre-inserted in the startup `Config` list (`elixir.erl:82-102`) and re-`put` during bootstrap (`elixir_compiler.erl:170-176`), so no caller violates the "key must exist" precondition. `update/2`/`get_and_put/2` (which internally use `get/1`) have no in-cluster callers.
- **Tokenizer/parser return shapes.** `elixir:string_to_tokens/5` matches the exact 6-tuple `{ok,…}` and 5-tuple `{error,…}` emitted by `elixir_tokenizer` (`elixir_tokenizer.erl:159,179,1918`); `string_to_quoted!` matches the `{Meta,Error,Token}` shape produced by `tokens_to_quoted` (`elixir.erl:470-473,522`).
- **Compiler ↔ code_server module-pool pairing.** `retrieve_compiler_module`/`return_compiler_module` (`elixir_compiler.erl:82,104,126-130`) map correctly to the `mod_pool` call/cast handlers and the `{{purged,…}}` monitor `handle_info` (`elixir_code_server.erl:65-71,102-105,129-132`).
- **Arities across files.** `elixir_aliases:expand_or_concat/4`, `elixir_module:compile/6` (from both `elixir_compiler.erl:156` and the `elixir_bootstrap.erl:28` AST), and `elixir:erl_eval/3` (`elixir_compiler.erl:59`) all match their definitions.
- **`bootstrap` flag & `identifier_tokenizer`** flow consistently through the `persistent_term` set by `elixir_config:static/1` across `elixir.erl:104`, `elixir_compiler.erl:169`, and the readers in `elixir_compiler.erl:40`/`elixir_aliases.erl:11`.
- **`#elixir_ex{}` `prematch`/`vars` record shape** is used compatibly between `elixir_clauses.erl` (`match/6`, `def/3`, `validate_cycles`) and `elixir_bitstring.erl:299-303` (`{Read, Cycle, Meta|{bitsize,…}}`), per the layout documented in `elixir.hrl:19-49`.
