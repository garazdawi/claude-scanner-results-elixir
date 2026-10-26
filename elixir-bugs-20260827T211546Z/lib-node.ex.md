# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/node.ex
<!-- sha256: 88a4ee2261c20036a1c1d9cf975c3925bf707c8b6867295752748d52ca0c6d0c -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/node.ex
<!-- sha256: 88a4ee2261c20036a1c1d9cf975c3925bf707c8b6867295752748d52ca0c6d0c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28 (hedge-clearing pass over the 2026-08-27 first pass)_

## Summary

`Node` is a thin, faithful wrapper over Erlang's `:erlang`/`:net_kernel`/`:net_adm` node functions. I found no genuine reachable bugs in its documented public API. In this pass I replaced the first pass's asserted/out-of-scope hedges with executed evidence: I ran the clause-dispatch and default-argument routing through the Erlang/OTP 29 toolchain in the sandbox, confirmed the OTP callees the module delegates to actually exist at the arities used, and ran the real `Node` functions at runtime. Every prior "Ruled out" entry is now backed by something I ran, with its axes enumerated; none had to be upgraded to a finding.

## Bugfixes

No findings

## Risks / caveats

- Every exported function is a one-line delegation to a BEAM built-in (`:erlang.node/0`, `:erlang.nodes/1`, `:net_kernel.start/1,2`, `:net_adm.ping/1`, etc.). The full *behavioural* semantics of those callees live in OTP and are that code's contract, not a reviewable assumption in this file. What I could check for this file without re-deriving OTP — that each delegation targets a function that exists at the arity used — I did: `net_kernel:module_info(exports)` on the sandbox's OTP 29 reports both `start/1` and `start/2`, so the `is_list` clause (`:net_kernel.start(name, Map.new(opts))` → `start/2`), the legacy atom clause (`:net_kernel.start([name, type, 15_000])` → `start/1`), and the arity-3 clause (`start/1`) all resolve to real functions. I did not re-derive net_kernel's option-map semantics.
- The deprecated `start(name, type)` / `start(name, type, tick_time)` clauses (marked `# TODO: Deprecate me on Elixir v1.23`) deliberately accept an atom second argument that the arity-2 `@spec` (keyword list) does not describe. This is intentional back-compat, not a defect, and is reachable only via the legacy signature; I confirmed by execution (below) that the legacy atom form routes to the legacy `net_kernel:start/1` list path and never collides with the keyword path. If Dialyzer strictness were the concern, the atom clause diverging from the `keyword`-typed spec is the only thing a tool would flag. Confidence: high that this is intentional, not a bug.

### Ruled out

- **`start/2` clause dispatch (keyword vs. deprecated atom form)** — `is_list(opts)` is checked before `is_atom(type)`, so no shape reaches the wrong clause. Basis: **verified**. Axes examined: default arity-1 call (`start(:foo)` → list clause, `opts=[]`); keyword list (`name_domain: :shortnames` → list clause); explicit empty list (→ list clause); legacy atoms `:longnames`/`:shortnames` (→ atom clause → `net_kernel:start/1`); `nil` second arg (→ atom clause, since `nil` is an atom, not a list — matches legacy semantics); arity-3 form (→ arity-3 clause); non-list/non-atom seconds — a map and a binary (both → `FunctionClauseError`, no wrong-clause fallthrough). Not examined: none. Ran: `/tmp/dispatch_test.exs` replicating the three clause heads with guards under `elixir` (OTP 29), observing which clause each input reached. Confidence: high.
- **`set_cookie(node \\ Node.self(), cookie)` default-argument binding** — Elixir fills defaults left-to-right, so `set_cookie(c)` yields `node = Node.self(), cookie = c`, matching the `@doc`. Basis: **verified**. Axes examined: single-arg call (arg binds to `cookie`, `node` takes the `Node.self()` default — `{:local@host, :mycookie}`); two-arg call (`{node, cookie}` positional); guard interaction with the leading default — a non-atom cookie in the single-arg form (`"str"`) raises `FunctionClauseError` rather than silently binding the string to `node`. Not examined: none. Ran: `/tmp/cookie_test.exs` replicating the exact head `def set_cookie(node \\ self(), cookie) when is_atom(cookie)`. Confidence: high.
- **`ping/1` doctest** — `Node.ping(:unknown_node)` returns `:pang` as documented. Basis: **verified**. Axes examined: actual runtime return of the real `Node.ping/1` on a non-existent node. Not examined: none for the doctest claim (the reachable-node `:pong` branch is OTP `net_adm` behaviour, out of this file's scope). Ran: `elixir -e 'IO.inspect(Node.ping(:unknown_node))'` → `:pang`. Confidence: high.
- **`spawn/3,5` `@spec` referencing `Process.spawn_opts()`** — the referenced type exists, so the spec is not dangling. Basis: **verified**. Axes examined: presence and definition of the type in the neighbour module. Not examined: none for the existence claim. Ran: Grep of `lib/elixir/lib/process.ex` → `465: @type spawn_opt ::`, `475: @type spawn_opts :: [spawn_opt]`. Confidence: high.
- **`@type state` declared after its use in `list/1`'s `@spec`** — module-level `@type` is order-independent at compile time; the module compiles and the delegation runs. Basis: **verified**. Axes examined: real runtime execution of the compiled function whose spec forward-references the later `@type`. Not examined: none. Ran: `elixir -e 'IO.inspect(Node.list())'` → `[]`, no compile warning. Confidence: high.

## Did I verify this?

- Clause dispatch and default-arg binding: **verified by execution** in the sandbox (OTP 29 / Elixir), replicating the exact clause heads and guards and observing routing across every input axis listed — not asserted from reading guards. `FunctionClauseError` outcomes for out-of-contract seconds were observed, not inferred.
- OTP callee existence: **verified** via `net_kernel:module_info(exports)` (`start/1` and `start/2` both present); OTP behavioural semantics beyond arity existence: **not verified** and intentionally left to OTP as that code's contract, not a hedged assumption about a file I could read.
- `Process.spawn_opts()` type: **verified** by reading the neighbour module `process.ex`.
- Runtime returns of `Node.list/0`, `Node.ping/1`, `Node.self/0`: **verified** by running the real module.

## Dropped in re-review

None. Every prior "Ruled out" entry survived and was strengthened from `asserted`/grep-only to executed evidence with its axes enumerated (`Not examined: none`). No prior finding existed to drop, and no hedge required upgrading to a low-confidence finding because each was cleared by something I ran rather than by re-reading a guard.
