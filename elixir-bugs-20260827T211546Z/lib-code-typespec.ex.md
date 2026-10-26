# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code/typespec.ex
<!-- sha256: bf405a7b31e749a6f81ec5a7368ea91530b6e8c39f81c5cf602a6ae601c3775a -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T21:56:36Z_

## Summary
`Code.Typespec` is byte-for-byte identical to the compiler's own installed build and converts types/specs/callbacks cleanly across every real BEAM module I threw at it, so I found no genuine reachable bug in its documented public API.

## Bugfixes

No findings

I verified the file against the Elixir build under review (`diff /opt/elixir/lib/elixir/lib/code/typespec.ex <target>` — no output, i.e. identical), then exercised the four documented public entry points (`fetch_types/1`, `fetch_specs/1`, `fetch_callbacks/1`, `spec_to_quoted/2`, `type_to_quoted/1`) against **222 loaded Elixir *and* Erlang modules** (all `:code.all_loaded/0` plus stdlib favourites), converting every type/spec/callback clause: `modules scanned: 222 / errors: 0`. The tricky shapes (structs, maps with `optional/required`, keyword lists, ranges, binaries `<<_::N>>`/`<<_::_*N>>`, any-arity funs, bounded funs with `is_subtype` constraints, free type variables) all round-tripped correctly, and the documented `:error` contract held for missing modules, non-BEAM binaries, and raw BEAM binaries.

## Risks / caveats
- The scan covered only types/specs/callbacks embedded in currently-loaded modules (222 of them, spanning Erlang OTP and Elixir stdlib). A pathological Erlang Abstract Format tuple not produced by any real compiler could still fall through a `case`/function clause (e.g. an exotic `bounded_fun` whose inner type is not `:product` would `badmatch` at line 36), but that is not reachable from a legitimately-compiled BEAM file, so it is out of scope per the documented-input contract.
- `@moduledoc false` marks the whole module compiler-private; every function nonetheless carries `@doc`/`@spec` and is `def`-exported, so I treated them as the documented public surface as instructed.
- I could not diff against the *canonical upstream* Elixir release independently — the only reference available in-sandbox is the build derived from this same source tree. The parity claim above is therefore "identical to the built compiler on this machine," not "identical to a third-party upstream."

### Ruled out
- **`erl_to_ex_var/1` using `String.to_unsafe_atom/1` (line 404/407) leaking/creating atoms** — not a bug: the function exists in this build (`function_exported?(String, :to_unsafe_atom, 1)` → `true`, verified) and unconditional atom creation is the intended behavior for reconstructing type-variable names from trusted BEAM data. Basis: verified.
- **`spec_to_quoted/2` free-variable branch (`{:when, meta, [spec, vars]}`, line 30) mis-threading `vars`** — exercised with `@spec ident(a) :: a when a: var` and a two-var case; output `ident(a) :: a when a: var` / `two(a, b) :: {a, b} when a: var, b: var` is correct. Basis: verified.
- **`fetch_specs`/`fetch_callbacks` `:error` clause vs `fetch_types` `_` catch-all asymmetry (lines 131/152 vs 110)** — `typespecs_abstract_code/1` only ever returns `{:ok, _}` or `:error`, so both spellings are total; missing/bad inputs returned `:error` as documented. Basis: verified.
- **`:binary` type ordering `{arg1,0}` vs `{0,arg2}` (lines 295–302)** — `<<_::8>>`, `<<_::_*4>>`, and `<<_::8, _::_*4>>` all round-tripped exactly. Basis: verified.
- **Map/struct handling via `List.keytake(:__struct__)` (line 281) accepting a wrong shape** — `%URI{}` and `%{required/optional => …}` types converted correctly; the `struct != nil and is_atom(struct)` guard gates the struct branch. Basis: verified.
- **`unpack_typespec_kw/2` mis-detecting keyword-list types (line 411)** — keyword type `[foo: integer, bar: atom]` produced `[foo: integer(), bar: atom()]`; non-keyword unions fall to the `[union]` form. Basis: verified.

