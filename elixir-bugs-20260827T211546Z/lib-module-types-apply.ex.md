# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/apply.ex
<!-- sha256: 8b9b72bb778ed51944c188e5d05bc27ec73251057140782effc4d8cd4ecb22b3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:48:51Z_

I've completed a thorough review with behavioral verification against the built compiler (confirmed byte-identical to the source under review). Here is my report.

## Summary

After a full read of `apply.ex` and behavioral testing of its tuple/map/comparison/capture code paths against the matching 1.21.0-dev compiler build, I found no genuine reachable bug; the tricky index arithmetic, domain handling, and error-formatting paths all check out.

## Bugfixes

No findings

## Risks / caveats

- This module is `@moduledoc false` compiler-internal code; its only reachable public path is "compile Elixir source with type checking on." I treated a crash or provably-wrong result on *valid* source as in-scope, since untrusted source text reaches these functions through the documented compiler entry point.
- I verified the mechanics that most plausibly hide a defect but could not exhaustively drive every branch (e.g. `:erlang.raise` stacktrace-shape rendering in `format_diagnostic`, the `:interpolation`/`:generator`/`:into` diagnostic banners, and every `remote_domain/7` key-specialized clause). These were reasoned through, not all executed.
- The diagnostic/`format_diagnostic` family is only invoked when an error has already been detected; a latent crash there would corrupt error *reporting* rather than mis-type valid code. I read all of it and found the `nil`-module case (`remote/7` → `remote_apply(info, nil, …)`) correctly guarded (`if mod`, `Code.ensure_loaded?(nil) == false`), but did not force-trigger every reason shape.

### Ruled out

- **Tuple index arithmetic off-by-one** across `element`/`insert_element`/`setelement`/`delete_element`/`Kernel.elem`/`put_elem`/`Tuple.insert_at`/`delete_at` (lines 469–665). I checked each `max(index±k, …)` and the corresponding `tuple_fetch`/`tuple_*_at(…, index-1|index)` offset against Erlang's 1-based vs Elixir's 0-based conventions, and the display index in each `:badindex`/`:negindex` error. All consistent. Basis: `verified` — compiled `elem/2`, `:erlang.element/2`, `put_elem/3`, `Tuple.insert_at/3`, `Tuple.delete_at/2` with tuple args; no crash, correct warnings (t2.ex).
- **`domain(nil, clauses)` returning `nil` for a multi-clause signature** (lines 1873–1874), which would make `zip_map_reduce(args, nil, …)` crash. I traced signature construction: `{:infer, nil, _}` and `{:strong, nil, _}` are produced *only* for single-clause signatures (`types.ex:322,382,437` gate `nil` on `[_] -> nil`; `apply.ex:309` likewise), so `nil` domain never pairs with 2+ clauses. Basis: `verified` (read `types.ex` domain-construction; both sites branch `[_] -> nil`).
- **`map_get` returning an unhandled `{:optional, _}` shape** into `Map.get`'s 3-clause `case` (lines 1196–1202). `Descr.map_get/2` (descr.ex:4026–4058) only ever returns `:badmap | :error | {:ok, _}`, so every consumer's `case` is total. Basis: `verified` (read `map_get/2`).
- **`Map.get`/`Map.fetch` warning "will always return nil/:error" on a statically-absent key** (lines 1174–1210). This fires for closed-map literals (`Map.get(%{a: 1}, :missing, false)` warns). It is the type system's deliberate closed-map / definitely-absent-key detection, applied uniformly across `fetch`/`fetch!`/`get`/`take`/`find`, not a logic defect in `apply.ex`. Basis: `verified` (t1.ex, t4.ex produce the designed warning; behavior is consistent across the family).
- **`:maps.from_keys` building `[List.delete(domain_keys, :atom) | atom_keys]`** (line 1384) — a list head spliced with atom-tail then fed to `closed_map`. Suspicious-looking but the `Enum.map` clause split (`is_atom(key)` vs list `key`) is intentional domain-key encoding. Basis: `verified` (t3.ex `:maps.from_keys` with literal, atom-var, and opaque-list args; no crash).
- **`Map.update`/`map_update_or_replace_lazy` `throw`/`catch` asymmetry and post-`try` rebinding of `map`** (lines 1317–1355, 1845–1869). The `catch {:badapply, reason, args_types}` shape matches the `throw` shape; `else` clauses reference the outer `map`/`fun` (Elixir `try` does not leak inner bindings), which is the intended fallback. Basis: `verified` (t3.ex `Map.pop_lazy`, `Map.replace_lazy` compiled cleanly) plus reasoning on `try` scoping.
- **`do_remote`/`remote/8` return-shape consistency** (2- vs 3-tuple, lines 396–406). Every `do_remote` clause returns either `{type, context}` or `{info, domain, context}`, both handled by the `case`. `:lists.member` (which can return either) and the default clause were checked explicitly. Basis: `asserted` — enumerated all `do_remote/8` clauses in the file; no execution path produced a mismatch in t2.ex.
- **`@doc` contract references** on `remote/7` ("remote_domain/4 + remote_apply/7"), `remote/8` ("remote_domain/7 + remote_apply/7"), and `return_compare` — each named helper exists at the stated arity. Basis: `verified` (arities present in-file).

