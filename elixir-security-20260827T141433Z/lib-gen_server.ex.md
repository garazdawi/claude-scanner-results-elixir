# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/gen_server.ex
<!-- sha256: 16860a5b7429a669bdca3add7a55d09de0cec04d8fa35aa0d37d8e943b73cfe2 -->
<!-- tree: 3389f8405 2026-08-25 -->

# Security review — `gen_server.ex` (re-review)

## /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/gen_server.ex
<!-- sha256: 16860a5b7429a669bdca3add7a55d09de0cec04d8fa35aa0d37d8e943b73cfe2 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28T:re-pass — hedge phrase "outside this file" resolved by reading callgraph neighbours (`:gen_server`, `:gen`) from the local OTP tree (`/usr/local/lib/erlang/lib/stdlib-8.0.1/src/`)._

## Summary
`GenServer` is a thin wrapper over Erlang's `:gen`/`:gen_server`/`:global`. Its documented public API contains no code-execution, atom-injection, deserialisation, path, or command sink reachable from untrusted input. The prior pass hedged three claims as "outside this file"; two are now **verified by reading the neighbour source** rather than assumed, and one remains a legitimate scope boundary (user callback modules) that introduces no unverified sink in *this* file. No security defect found. No findings upgraded, none dropped.

## Security

`No findings`

### Ruled out

- **`{:via, mod, name}` module dispatch in `cast/2` and `whereis/1`** — `cast({:via, mod, name}, req)` calls `mod.send/2` (line 1215) and `whereis({:via, mod, name})` calls `apply(mod, :whereis_name, [name])` (line 1355), invoking an arbitrary module/function. *Basis: verified* — a `Spy` module's `send/2` and `whereis_name/1` were both invoked (executed against Elixir 1.21.0-dev / OTP 29 in-tree build). Not a finding: `mod` is a component of the caller-supplied `server`/`name` value, whose documented type is `{:via, module, term}` (in-file spec lines 243–245, 866). This is the OTP `:via` registry contract, not a value any documented API forwards from network/disk/dependency data. The identical dispatch exists in the neighbour `:gen_server.cast/2` (`gen_server.erl:1812` — `cast({via, Mod, Name}, Request)`), confirming this is the intended registry protocol, not an Elixir-side amplification.\
  Axes examined: source of `mod` (caller-constructed server ref, documented type), dispatch reachability (executed). Not examined: none.

- **Default injected `handle_info/2` logging unexpected messages (lines 944–968), formatted by `format_report/1` (line 1374)** — a module using the default (no `handle_info`) logs every unsolicited message. *Basis: verified* — sending `{:evil, "\n*DBG* forged log line", :"$secret"}` produced a single escaped log line, not a forged newline; `format_report/1` passes the message as a `~p` argument (`~c"~p ~p received unexpected message in handle_info/2: ~p~n"`, line 1374), so the term printer escapes control characters. No log injection. DoS relevance is negligible (any process is reachable by `send/2`) and the behaviour is documented (line 700).\
  Axes examined: newline/control-char injection via `~p` (executed), documentation (in-file). Not examined: none.

- **Distributed dispatch (`abcast/3`, `multi_call/4`, `{name, node}` in `call`/`cast`/`whereis`)** — **Upgraded verification (was hedged "no parsing of untrusted bytes occurs in this file" in first pass; now read the neighbour).** `multi_call/4` (line 1284) delegates to `:gen_server.multi_call/4`. Reading `gen_server.erl:2005–2064`: the implementation (`mc_send/6`, `mc_recv/5`) only performs `erlang:monitor(process, {Name, Node})` and `erlang:send({Name,Node}, {'$gen_call', ...}, [noconnect])` over caller-supplied node atoms — pure message-passing and monitor bookkeeping. There is **no byte parsing, no atom synthesis from external data, and no code-execution sink** in the delegated path. `abcast/3` (line 1239) reduces to `do_send({name, node}, msg)` per node, same `send/2` primitive. The "outside this file" caveat is now discharged by reading the callee, not assumed.\
  Axes examined: delegated callee behaviour (read `gen_server.erl:2005–2064`), atom/byte-parsing sinks in callee (absent), node atom provenance (caller-supplied). Not examined: none.

- **Atom creation from external data** — none present. No `String.to_atom`, `binary_to_atom`, `List.to_atom`, `Module.concat`, or similar in the file; `:name`/`server` atoms are consumed, never synthesised from bytes. CWE-770 atom-table exhaustion does not apply here.\
  Axes examined: full-file search for atom-synthesis primitives. Not examined: none.

- **Compile-time execution via `__using__/1` (lines 899–1000)** — `opts` are the developer's own `use GenServer, ...` arguments, `Macro.escape/1`d into the generated `child_spec/1` and forwarded to `Supervisor.child_spec/2`; no filesystem/network access at build time, no untrusted-input path.\
  Axes examined: input provenance (developer's own source), sinks in generated body. Not examined: none.

- **`__before_compile__/1` (lines 1002–1029)** — compile-time `IO.warn` interpolating `inspect(env.module)` (the module being compiled); no external input, no injection.\
  Axes examined: input provenance. Not examined: none.

- **TOCTOU between `whereis/1` and `stop/3`/`call/3`** — both look up the server then act on the result (lines 1132, 1174); the race is inherent, explicitly documented ("no guarantee the returned `pid` ... is alive", line 1326), and the subsequent `:gen.call`/`:proc_lib.stop` exits are caught and converted to `exit/1` (lines 1140–1145, 1179–1183). No exploitable state corruption.\
  Axes examined: race window, error handling of stale pid. Not examined: none.

- **`cast/2`/`do_send/2` swallowing all errors with `catch _, _ -> :ok` (lines 1208–1209, 1217–1218, 1251–1252)** — documented fire-and-forget contract ("always returns `:ok` regardless of whether the destination ... exists", lines 1193–1196), not a silent-failure security defect.\
  Axes examined: documented contract vs. behaviour. Not examined: none.

## Risks / caveats
- **Genuine scope boundary (not an unverified sink):** `GenServer` delegates the message loop, name registration, and `init/1`/`handle_*` invocation to `:gen`/`:gen_server` and to the *user's callback module*. A specific application's callback (e.g. a `handle_call` that evaluates its `request`, or a custom `:via` registry with an unsafe `whereis_name`) is that application's code. This file introduces no sink whose safety depends on unread behaviour there — the two previously-hedged delegated paths (`:via` dispatch, distributed `multi_call`/`abcast`) were read in the neighbour source this pass and confirmed sink-free.
- The `{:via, mod, ...}` / `{:global, ...}` paths are safe *given the documented in-file contract* (spec lines 243–245, 866) that the server reference is caller-supplied, not attacker-supplied. An application that builds `{:via, module, term}` from untrusted input turns dispatch into arbitrary-module invocation — a call-site misuse, out of scope for this file.
- Behavioural claims were reproduced against an `elixir`/`erl` build from this tree (Elixir 1.21.0-dev, OTP 29); neighbour claims were verified by reading `stdlib-8.0.1/src/gen_server.erl` and `gen.erl`.

## Did I verify this
- **`:via` dispatch reaches arbitrary `mod`** — yes, executed (Spy module `send/2`/`whereis_name/1` invoked).
- **Default `handle_info` logging cannot inject newlines** — yes, executed (`~p` escaped the forged payload).
- **Distributed `multi_call`/`abcast` delegate to a sink-free path** — yes, read the neighbour `:gen_server.multi_call/4` + `mc_send/6`/`mc_recv/5` (`gen_server.erl:2005–2064`) and `cast/2` (`gen_server.erl:1809–1819`); confirmed pure message-passing, no byte parsing / atom synthesis / eval.
- **No atom synthesis from bytes** — yes, full-file search.

## Dropped in re-review
None. All prior ruled-out entries remain correct; two had their "outside this file" hedge discharged by reading the neighbour source, and none were upgraded to findings because no in-file sink depends on unverified callee behaviour.
