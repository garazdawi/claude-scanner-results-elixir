# /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/iex.erl
<!-- sha256: fd7de6d570d31be729f578a899970d75646d9e70ff64c37548f6d99707f97085 -->

_Analysed on 2026-08-27T17:33:13Z_

## Summary

`iex.erl` is a thin CLI/boot shim (start the IEx shell, parse `--dot-iex`/`--remsh` from the VM's plain arguments, wire up an optional remote shell); every input it handles is operator-controlled command-line argv or a trusted embedding caller's `Opts`/`MFA`, so no attacker trust boundary is crossed and there are no findings.

## Security

No findings.

The file exports `start/0`, `start/2`, `shell/0`, `sync_remote/2`. Its only external inputs are:

- `init:get_plain_arguments()` (`iex.erl:52`) — the Erlang VM's command-line arguments, i.e. whatever the operator typed when launching `iex`. Parsed by `get_dot_iex/1` (`:81-83`) and `get_remsh/1` (`:85-87`).
- `Opts`/`MFA` passed to `start/2` (`:37`) — supplied by the in-folder caller (`elixir:start/0` → `iex:shell/0` builds them from argv; `iex:start/0` hard-codes `{elixir_utils, noop, []}`) or by a program embedding IEx.

The threat model's attacker is *a supplier of untrusted Elixir source or an untrusted quoted AST*. That attacker does not control `iex`'s process argv nor the `MFA` an embedder passes to `start/2`. This is the same "developer-CLI argv" surface ruled out for the Mix profiler wrappers in prior reviews — no network, file-format, or deserialization input reaches this file. The Unicode identifier-security obligations in the standards checklist (bidi/`?break`, NFC, mixed-script, `\u` escapes) belong to `elixir_tokenizer.erl`/`elixir_interpolation.erl`; this file contains none of that machinery.

### Ruled out

- **`elixir_utils:characters_to_binary(H)` in `get_dot_iex/1` (`iex.erl:81`)** — the `--dot-iex` path.
Reason — `H` is the argv element after `--dot-iex`; operator-controlled, never attacker-supplied. The callee surfaces bad UTF-8 as a raised `UnicodeConversionError` rather than emitting garbage (`elixir_utils.erl:148-155`), so even a malformed value only self-crashes the operator's own boot.
Axes examined: valid UTF-8, invalid UTF-8 (`{error,_,_}`), incomplete sequence (`{incomplete,_,_}`), already-binary input | Not examined: none.
Basis — verified.

- **`--dot-iex` path → `{dot_iex, …}` opt → later evaluated as code (`iex.erl:78,81`)** — code-execution sink (in `Elixir.IEx.Server`, out of this file).
Reason — the operator names a config file to load and evaluate, exactly like `.bashrc`/`.iex.exs`; the file's contents are operator-trusted, and the eval happens in a different module. No trust boundary is crossed here.
Axes examined: path present/absent, path as last arg with no value (falls through to `nil` via `:82-83`), relative/absolute path | Not examined: none.
Basis — verified (parsing + fall-through read in this file; downstream eval is by-design and out-of-file).

- **`'Elixir.IEx.Server':run_from_shell(Opts, MFA)` in `start/2` (`iex.erl:48`)** — MFA dispatch / code execution.
Reason — `MFA` originates from internal builders (`start_mfa/2` → `{elixir,start_cli,[]}` or `{?MODULE,sync_remote,[…]}`; `start/0` → `{elixir_utils,noop,[]}`) or from an embedder calling the documented `start/2` API. An embedder passing a hostile `MFA` already has BEAM code execution and gains nothing new; a plain caller cannot influence it.
Axes examined: MFA from each internal builder, embedder-supplied MFA (uplift test) | Not examined: none.
Basis — verified.

- **`--remsh` `Remote` → `{remote, Remote, MFA}` → `user_drv:start` → remote-node connect (`iex.erl:70`, consumed at `elixir.erl:180`)** — network / node-name handling.
Reason — `Remote` is an operator-supplied node name; connecting to a distributed node is gated by the Erlang distribution cookie, the standard trust model. One node-name value per boot, operator-chosen; no atom-exhaustion (single conversion) and no attacker control of the target.
Axes examined: node name present/absent, remsh as last arg with no value (`get_remsh` fall-through to `nil`, `:86-87`), arbitrary node string | Not examined: none.
Basis — verified (this file + caller `elixir.erl:179-180`); distribution-cookie enforcement is OTP-standard.

- **`sync_remote/2` message passing (`iex.erl:73-75`)** — exported function / concurrency.
Reason — it sends `{'begin', Ref, self()}` to `Parent` and blocks on `receive {done, Ref}`. `Ref` is an unforgeable `make_ref()`. An arbitrary caller can only make its *own* process send one message and then block; it cannot unblock another process's `receive` without the matching `Ref`.
Axes examined: arbitrary `Parent` pid, arbitrary `Ref`, self-DoS by blocking | Not examined: none.
Basis — verified.

- **`spawn`/`spawn_link` (`iex.erl:41,62`)** — concurrency.
Reason — the spawned closures capture only internal values (an unforgeable `Ref`, hard-coded MFAs, `self()`); no external data influences scheduling or target, and the linked helper only relays a local `Ref` handshake.
Axes examined: captured-variable provenance, link-propagation effect | Not examined: none.
Basis — verified.

- **`get_dot_iex/1` / `get_remsh/1` argv walk (`iex.erl:81-87`)** — resource/robustness (crash on malformed argv).
Reason — each clause consumes one list element (`[_ | T]`) and terminates at `[]`; the `["--flag"]`-with-no-value case falls through the two-element head to the single-element head and returns `nil`. Argv is finite, so no unbounded loop and no unmatched-clause crash.
Axes examined: flag absent, flag last with no value, flag mid-list, empty list, repeated flags (first match wins) | Not examined: none.
Basis — verified.

### Did I verify this?

- `characters_to_binary` sink — Basis `verified`: I read `elixir_utils.erl:145-155` and confirmed all three result shapes (binary / `{error,…}` / `{incomplete,…}`) are handled, the last two by raising. `Not examined: none`. No `Depends on:` needed.
- `--dot-iex` code-exec, `run_from_shell` MFA, `sync_remote`, `spawn`, argv-walk — Basis `verified` from this file's own code (`iex.erl:34-88`), read in full; each rests on values that are internal or operator-argv, established without appeal to unread code. `Not examined: none`.
- `--remsh` remote-connect — Basis `verified` for the in-file data path (`iex.erl:54-71`) and its single in-folder consumer (`elixir.erl:179-180`, read). The residual assumption is external and standard: **Depends on: OTP `net_kernel`/distribution — remote-node connection is authenticated by the shared distribution cookie**, not by `iex.erl`. This is the documented Erlang distribution trust model, not a per-file contract I can tighten from here; it does not upgrade the sink because the node name is operator-supplied regardless.

## Risks / caveats

- **Operator trust is assumed and appropriate.** Anyone who can set `iex`'s command line (`--dot-iex FILE`, `--remsh NODE`, or the embedder's `MFA`) can run arbitrary code — by design, identical to invoking `iex` interactively. If a *higher-level* tool builds an `iex` command line from untrusted data and shells out, the command-injection defect lives in that tool, not in `iex.erl`, which receives already-parsed plain arguments.
- **`start/2` is a documented code-execution API**, not a sandbox: it runs the supplied `MFA`. Passing a hostile `MFA` requires the caller to already possess BEAM code execution, so it grants no new capability and is out of scope.
- **Cross-file dependence:** the only ruling that leans on code outside this folder's read set is `--remsh`, and it leans on OTP distribution's cookie authentication (standard, not `iex`-specific). Actual evaluation of the `--dot-iex` file happens in `Elixir.IEx.Server` (an Elixir-side module not in this Erlang src folder); if that module were to evaluate a *default*/auto-discovered `.iex.exs` from an untrusted working directory, that would be a finding in *that* file — nothing in `iex.erl` selects such a default (it returns `nil` when `--dot-iex` is absent).
- **Unicode standards obligations** in the checklist (bidi rejection, NFC, mixed-script, escape validation) do not apply to this file; it performs no tokenizing, identifier handling, or escape decoding.

