# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/utils.ex
<!-- sha256: 735e14078a81dc497fb185892d68351f1d072ee844f71bfbb6a229b229ac3946 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:33:58Z_

I've thoroughly reviewed and behaviorally tested every public entry point of this file. Here is my report.

## Summary

`Kernel.Utils` faithfully implements the compiler callbacks for `destructure`, `defdelegate`, `defstruct`, `raise`, and `defguard`; behavioral testing of every public entry point found no genuine, reachable bug.

## Bugfixes

No findings

### Ruled out

- **`unquote_refs_once`/`all_used` comprehension dropping `nil` for unused args (utils.ex:404)** — I suspected `for ref <- ..., used = :maps.get(ref, used_refs, nil), do: used` would leave `nil` entries (for guard args not used in the body) that then crash `:lists.unzip/1`. Verified this is *not* a bug: an assignment clause in an Elixir `for` acts as a truthiness filter, so `nil` values are dropped. `for k <- [:a,:b,:c], used = Map.get(%{a: 1}, k, nil), do: used` returns `[1]` (Basis: verified). End-to-end, `defguard both(a, b) when is_integer(a)` compiles and `both(3, 999)` returns true (Basis: verified).
- **`String.to_unsafe_atom/1` (utils.ex:391) being a nonexistent function** — verified it is a real, exported function throughout this fork (`lib/string.ex:3040`, with `@spec` and used across `base.ex`, `macro.ex`, etc.); this fork renamed the old `to_atom` to `to_unsafe_atom`. Not a bug (Basis: verified via Grep + successful compilation of defguard which calls it).
- **`destructure/2` non-termination when the source list is longer than `count`** — checked the clause ordering: the `destructure_list(_, 0)` head (line 21) precedes the `[h|t]` head, so an over-long list terminates at count 0. `destructure([a,b,c],[1,2])` yields `{1,2,nil}` and over-long inputs truncate correctly (Basis: verified).
- **`defguard` single-evaluation contract outside guards (utils.ex:378-411)** — the `@doc` promises the argument is unquoted once and reused. Verified: `G.is_even((IO.puts("side-effect"); 4))` prints `side-effect` exactly once and returns `true`; multi-arg `triple(pa.(),3,4)` evaluates its side-effecting arg once (Basis: verified).
- **`defdelegate` `:append_first` arg reordering (utils.ex:83-90)** — `tl(as_args) ++ [hd(as_args)]` correctly relocates the first arg to the end per the documented semantics; delegation, `:as`, and self-call detection all behave as documented (Basis: verified via working `Deleg.add/2`, `Deleg.myrev/3`, and the self-call ArgumentError).
- **`defstruct` enforce-keys / duplicate-key / non-atom-field handling (utils.ex:107-233)** — `@enforce_keys` enforcement, single-key wrapping, undefined-enforce-key rejection, and struct construction all produce the documented errors and values (Basis: verified: `struct(S1, a: 1)` → `%S1{a: 1, b: nil, c: 3}`; missing enforce key raises the documented message).
- **`raise/1` callback shapes (utils.ex:261-278)** — binary→RuntimeError, atom→`module.exception([])`, exception passthrough, and the catch-all ArgumentError all return the documented shapes (Basis: verified).
- **`@spec defguard/3` vs implementation (utils.ex:330)** — spec `([Macro.t()], Macro.t(), Macro.Env.t()) :: Macro.t()` matches the function returning a quoted AST (Basis: asserted from reading; the return is a `quote` block).
- **`defguard/2` `@doc` example (utils.ex:304-323)** — it is illustrative prose ("would print a code similar to"), not an `iex>` doctest, so it is not an executable contract (Basis: verified — no `iex>` prompt present, module is `@moduledoc false`).

## Risks / caveats

- The module is `@moduledoc false`; its "public API" is only the surface reached through the `Kernel` macros (`destructure/2`, `defdelegate/2`, `defstruct/2`, `raise/1`, `defguard`). I scoped the review to those documented paths and exercised each. I did not attempt to fuzz the compiler-internal `:elixir_module`/`:ets`/`:elixir_quote` interactions beyond confirming the documented macro behaviors, since those are internal contracts, not public entry points.
- All behavioral claims were run against the `elixir` built from this source tree (Elixir 1.21.0-dev / OTP 29) with `ELIXIR_ERL_OPTIONS="+fnu"` to satisfy the UTF-8 locale requirement.

