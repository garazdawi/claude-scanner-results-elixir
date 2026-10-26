# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/enum.ex
<!-- sha256: 605d3b9d49cf3b22a4798fe2aea566e24e3a396b556d05aab30d84b4f8d0f8ff -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:40:26Z_

All reproductions confirm my analysis. Let me record what I verified:

1. **Reservoir sampling** (`take_random/2`, Algorithm L) handles large count/collection with no arithmetic crash (`:math.log(1 - w)` stays in-domain): 5000 unique elements returned from a 1M range.
2. **Randomness is `:rand`-based and fully deterministic under a fixed seed** — two runs with the same seed produced byte-identical output `{494, [1,2,4,3,5], ~c"Y\r>"}`, confirming these are non-cryptographic (documented).
3. **Deprecated `group_by/3` Dict path** raises `UndefinedFunctionError`, atomizing only the fixed literal `"Dict"` — no user data reaches the atom table.

I have enough to write the report.

---

# Security review — `lib/elixir/lib/enum.ex`

## Summary
`Enum`/`Enumerable` is a pure functional collection library with no code-eval, command, file, network, deserialization, or shared-state sinks; the only externally-influenced dispatch points are contract-level (caller-selected sorter modules and protocol `__MODULE__`), so no security defect is reachable through its documented public API.

## Security

`No findings`

### Ruled out

- **Non-cryptographic randomness — `random/1`, `shuffle/1`, `take_random/2`, `random/1` on ranges (lines 2484–2530, 3018–3025, 3782–3883).** These call `:rand.uniform/0,1`, which is a fast PRNG, not a CSPRNG. This is the documented contract ("This function uses Erlang's `:rand` module") — the functions never claim cryptographic strength, so no security *property they are meant to provide* is weakened. Using them to mint secrets/tokens is caller misuse, not a defect here. Basis: `verified` — reseeding `:rand` with `{100,101,102}` produced byte-identical `random`/`shuffle`/`take_random` output across two separate VM runs, confirming predictability by design. Integer selection uses `:rand.uniform(n) - 1` (range `0..n-1`), so there is no modulo bias. (Residual risk noted below.)
- **Atom creation `String.to_unsafe_atom("Dict")` (line 1445), deprecated `group_by/3`.** The argument is a compile-time string *literal*, so at most one fixed atom (`Dict`) is ever created regardless of call count — no atom-table growth from external input (CWE-770 does not apply). The enumerable elements/keys flowing through `dict_module.update/4` are used as *map keys*, never atomized. Basis: `verified` — `Enum.group_by([1,2,3], %{}, fn x -> x end)` raises `UndefinedFunctionError` (the `Dict` module was removed years ago) without atomizing any argument.
- **Dynamic module dispatch — `module.compare/2` (lines 1931, 2116, 2396, 3324–3326) and `module.reduce/3` (lines 725, 1021, 2041, 4486, 4785, 4801, 4853).** For `compare/2`, `module` is the caller-supplied `sorter` argument, documented as a `module()` the developer selects (e.g. `Date`); routing untrusted input into that slot is API misuse that violates the documented contract, and it only invokes `compare/2` on an already-loaded module. For `reduce/3`, `module` is the `{:error, __MODULE__}` returned by the `Enumerable` implementation already resolved from the enumerable's *type*, i.e. standard protocol dispatch — an attacker would need to already have a malicious module loaded (code execution) to abuse it. Neither crosses a trust boundary reachable from data alone. Basis: `asserted` (control-flow/contract reasoning; the module names are structurally never derived from element values).
- **`join/2` and `map_join/3` via `entry_to_string` → `String.Chars.to_string` (lines 1682–1704, 1824–1828, 4296–4297).** `#{}`-style stringification happens, but the output is a plain binary/iodata; there is no downstream interpreter (SQL/HTML/shell/regex) in this file. Escaping for a target sink is the responsibility of whatever consumes the returned string, which is out of scope for this module. A non-`String.Chars` element raises, as documented. Basis: `asserted`.
- **Resource consumption / algorithmic complexity.** All traversals are the documented linear/eager behavior (the moduledoc explicitly warns about infinite enumerables and directs users to `Stream`); the caller controls input size. Spot-checked non-obvious paths — `flat_map`/`concat` right-associated `++` are O(total), merge `sort` is O(n log n), `take_random` reservoir sampling does not amplify a huge `count` against a small collection (`take_mapped(sample, min(count, size), …)`), and `chunk_every/4` consumes only `count - length` elements from an infinite `leftover`. Basis: `verified` for reservoir sampling (5000 uniq elements from `1..1_000_000` with no crash); `asserted` for the others (source-level complexity reasoning).
- **`member?/2`, `all?`, `any?`, `empty?` predicates.** These are collection predicates, not documented safety/authorization gates, and use well-specified strict `===/2` semantics (e.g. `2 !== 2.0`). No hidden wrong-answer path that would silently return "safe." Basis: `asserted` (matches documented `===` contract).
- **`into_protocol` re-raise (`:erlang.raise(kind, reason, __STACKTRACE__)`, lines 1569, 1640).** This is correct clean-up-then-reraise on a caught `Collectable` failure, preserving the original stacktrace; it neither swallows errors nor leaks state. Basis: `asserted`.

## Risks / caveats
- **Residual risk (developer-facing, not a code defect):** `Enum.random/1`, `Enum.shuffle/1`, and `Enum.take_random/2` are predictable (verified deterministic under a fixed `:rand` seed). If downstream code uses them to generate session IDs, tokens, password-reset nonces, shuffle for a "fair" draw, or any security-sensitive selection, that is exploitable at the *application* layer. The fix belongs to callers (use `:crypto.strong_rand_bytes/1`), not this file, but it is the single most likely way `Enum` participates in a real vulnerability.
- I reviewed only `enum.ex`. Behavior of collaborators it dispatches into — `String.Chars`/`Collectable`/`Enumerable` implementations, `List.keysort/3`, `Range.size/1`, `MapSet` — was treated as trusted per their own contracts and not audited here.
- Dynamic-dispatch and complexity conclusions are contract/source-reasoning based (`asserted`); I verified the randomness, reservoir-sampling, and deprecated-`Dict` claims by execution against the in-tree Elixir 1.21.0-dev build. No path in this file converts external bytes to atoms, evaluates code, touches the filesystem/network, or writes global mutable state.

