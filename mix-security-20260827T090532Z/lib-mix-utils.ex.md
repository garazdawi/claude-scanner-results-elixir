# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/utils.ex
<!-- sha256: 535e8b2f1e39579e397f1f8c4c3de97734a839dc3d23d47b9af2a65e2dac832f -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/utils.ex
<!-- sha256: 535e8b2f1e39579e397f1f8c4c3de97734a839dc3d23d47b9af2a65e2dac832f -->

_Re-reviewed on 2026-08-27_

The prior report's central finding hedged on the reachability of `non_executable_binary_to_term/2` through `Mix.Sync.PubSub` — flagged with the phrase "outside this file." I read `elixir/lib/mix/lib/mix/sync/pubsub.ex` in full and re-ran the interning/`:safe` behavior. The reachability, the loopback/no-auth transport, and the unbounded 64-bit size gate are now **verified from source**, so the finding is retained with its evidence upgraded. One caveat (terminal-escape via tree printers) could not be verified from any neighbour available to me and is upgraded to a low-confidence finding per the re-review rule.

## Summary
`Mix.Utils`'s documented "restricted" deserializer `non_executable_binary_to_term/2` defaults to a non-`:safe` `:erlang.binary_to_term/2`. It is the *only* decode path for frames arriving on `Mix.Sync.PubSub`'s unauthenticated loopback TCP socket, and decoding happens **before** any hash/subscription check, so any local process that reaches the ephemeral port can intern unlimited atoms and eventually abort the build VM (and, via the same sink, force unbounded term expansion).

## Security

#### `CSCAN-utils-binary-to-term-atom-exhaustion`
- **Finding ID** — `CSCAN-utils-binary-to-term-atom-exhaustion`
- **Severity** — Medium
- **Confidence** — medium
- **Location** — `elixir/lib/mix/lib/mix/utils.ex:30-34` (`non_executable_binary_to_term/2`; sink at line 31, `:erlang.binary_to_term(binary, opts)` with `opts` defaulting to `[]`)
- **Class** — Deserialisation / Atom creation (CWE-502, CWE-770)
- **Impact** — The function is documented (utils.ex:14-27) as a *restricted* `binary_to_term`, but its default `opts` omit `:safe`, so it interns every atom present in the input. `Mix.Sync.PubSub.decode_data/1` calls it with the default (`sync/pubsub.ex:266`, `Mix.Utils.non_executable_binary_to_term(binary)` — no `[:safe]`). **Verified reachability (read `sync/pubsub.ex` in full):**
  - The subscriber listens on `@loopback = {127,0,0,1}` via `:gen_tcp.listen(0, @listen_opts)` (`listen/0`, lines 26-27, 207-218) — an ephemeral loopback port, `active: false`, no authentication of any kind.
  - A spawned `receive_message_loop/2` (lines 197, 231-235) accepts any connection and, in `receive_message/1` (lines 237-262), reads a frame `<<"mixpubsub", hash::binary-size(16), size::unsigned-integer-64>>` (line 246) then `recv(socket, size)` (line 248) then `decode_data(binary)` (line 249).
  - **Decode precedes the subscription check:** `decode_data/1` interns atoms at line 266; the `hash_to_pids[hash]` filter only runs later, in `handle_info/2` at `sync/pubsub.ex:161-162`. So atom interning occurs for *any* structurally valid frame, even one whose hash matches no live subscription — the attacker need not know the subscription key.
  - Port discovery: subscribers write `port_<port>` files under `System.tmp_dir!()/mix_pubsub_user<uid>/<base64(md5 key)>/` (`base_path/0` line 279-282, `create_subscription_file/2` lines 220-224). A same-user co-resident process lists the dir directly; a cross-user or blind attacker scans loopback. Either way the port is reachable.
  - During `mix compile` the pubsub is the compile-invalidation channel, so the listener is live on the build path.

  Atoms are never garbage-collected; sustained unique-atom frames grow the atom table until it hits the limit and the build VM aborts — a local, cross-user DoS on the build path. **No code-execution angle:** `non_executable_terms/1` (utils.ex:36-63) rejects funs, and `[:safe]` would too.
- **Evidence** — `verified` (reachability), `verified` (interning + `:safe` rejection), `asserted` (final table-full abort).
  - Re-ran a hand-crafted ETF for a brand-new atom (`SMALL_ATOM_UTF8_EXT`, tag 119) through `binary_to_term/2` with default `[]`: `term=zzq_1122 delta=1` — a never-before-interned atom is created. With `[safe]` an unknown atom → `{error,badarg}`. (OTP toolchain in sandbox, this pass.)
  - Prior measured interning rate retained: one ~868 KB list payload interned 50 000 atoms against the default `atom_limit` of 1 048 576 ⇒ ≈21 messages (or one ~18 MB message) exhaust the table. The `size` field is a full unsigned 64-bit (`sync/pubsub.ex:246`, `recv(socket, size)` at 248 with **no upper bound**), so a single oversized frame is admissible.
  - Fun rejection is complete (prior pass): bare fun / fun-in-tuple / -list / -map-value all rejected; plain data accepted.
  - I did not exhaust the host VM's atom table (would crash the sandbox); the final `no more index entries in atom_tab` abort is documented OTP behavior → `asserted`. Interning rate and `:safe`/fun rejection are measured.
- **Suggested fix** — Default to the safe form (`opts \\ [:safe]`) or have the untrusted caller pass `[:safe]` at `sync/pubsub.ex:266`. With `:safe`, unknown atoms raise `ArgumentError`, which `decode_data/1`'s `rescue` (lines 267-268) already turns into `:error` and drops. The atoms Mix legitimately broadcasts are pre-existing (module/message-tag atoms), so `:safe` does not break real traffic.

#### `CSCAN-utils-tree-printer-terminal-escape` — Upgraded from hedged assumption in first pass
- **Finding ID** — `CSCAN-utils-tree-printer-terminal-escape`
- **Severity** — Low
- **Confidence** — low
- **Location** — `elixir/lib/mix/lib/mix/utils.ex:518` (`print_each_node/5`) and the JSON/DOT tree writers
- **Class** — Output neutralization / terminal-escape injection (CWE-150 / CWE-116)
- **Impact** — `print_each_node/5` and the JSON/DOT writers interpolate callback-supplied `name`/`info` straight into `Mix.shell().info` without stripping ANSI/control sequences. If a caller (`deps.tree`, `xref graph`, etc.) forwards an attacker-influenced dependency name/version (e.g. a string sourced from `mix.lock`/registry metadata), the escape sequences reach the terminal.
- **Evidence** — `asserted`. The tree callbacks are supplied by task modules **not in this file and not among the neighbour modules available to me** (only `sync/pubsub.ex` was in the callgraph context). I could not trace a concrete attacker-controlled string from an external caller into these printers, so I cannot verify or refute it here. Per the re-review rule (verify or upgrade), the former caveat is recorded as a low-confidence finding rather than dropped. Escaping *within* this file is limited to DOT quoting (`escape_dot_string`, see Ruled out); ANSI/control bytes are not stripped on any path.
- **Suggested fix** — Sanitize control/ANSI bytes in `name`/`info` before emitting, or confirm at each call site that these fields are constrained to a safe charset.

### Ruled out
- **`read_httpc` TLS retries (`utils.ex:906-925`)** — `verified`. Re-read: both retry branches reuse `http_options`/`ssl_options`, which carries `verify: :verify_peer` + hostname check. The ipfamily-fallback branch (lines 908-912) only swaps `ipfamily`; the `:tls_alert` branch (914-917) sets `middlebox_comp_mode: false` via `put_in(http_options, [:ssl, :middlebox_comp_mode], false)`, which toggles only the TLS 1.3 dummy-CCS, never certificate verification. No downgrade. *Axes examined:* retry-branch ssl option mutation, hostname-check retention. *Not examined:* none.
- **Checksum comparison (`checksum/1`, `hexhash/2`, `utils.ex:832-852`)** — logic correct and applied before bytes are returned; `--sha512` opt-in is documented/by-design. `actual != expected` is non-constant-time, but a SHA-512 integrity checksum is not a secret, so timing leaks nothing. *Axes examined:* order-of-check, timing-side-channel-secrecy. *Not examined:* none.
- **`parse_mfa/1` and `command_to_module/2` (`Code.string_to_quoted`, `Module.concat`)** — `string_to_quoted` parses without evaluating; the atom-creating `Module.concat` / MFA atoms come from developer-typed CLI args / task names (operator-controlled, single-shot), not from a dependency or the network. *Axes examined:* eval-vs-parse, atom-source provenance. *Not examined:* none.
- **`write_dot_graph!/5` DOT escaping (`escape_dot_string`, `utils.ex:602-623`)** — verified robust (prior pass): a node name `evil" ]; malicious [shape=box]; "x` and a trailing-backslash title stay inside the quoted string; no attribute/node injection. (Note: this escapes DOT metacharacters only, not ANSI/control bytes — see `CSCAN-utils-tree-printer-terminal-escape`.)
- **`write_according_to_opts!/3` `.bak` rename (`utils.ex:393-409`)** — output path is the operator's `--output`; any traversal/clobber is self-inflicted.
- **Duplicate `to_erl_term(map)` clause (`utils.ex:1087-1096`)** — dead (identical clause at 1076 handles all maps); a functional duplicate, not a bypassed defensive check. `:io_lib.print` quotes/escapes atoms and strings, so consult round-trips are not injectable.
- **`detect_user_id!/0` temp file (`utils.ex:1023-1048`)** — name uses `strong_rand_bytes` + `os_time`, result cached; shared-tmp symlink race is impractical and only mis-reports a UID.
- **`symlink_or_copy` / `read_file` / file ops** — operate on `_build`-derived paths; no attacker-controlled path component originates in this file.

## Risks / caveats
- **`read_httpc` global logging mutation (`utils.ex:903-923`)** — `Logger.configure(level: :error)` restored in `after`; two concurrent fetches can leave the global level stuck at `:error`. Not attacker-controlled; robustness/observability nit.
- **Resource amplification via the same pubsub sink** — now with the transport verified: `sync/pubsub.ex:246-248` admits a full unsigned-64-bit `size` and `recv`s that many bytes, then hands them to the `:safe`-less `binary_to_term`, so a compressed ETF payload can expand enormously. This is the same root cause as the finding (no `:safe`); the missing size cap lives in `sync/pubsub.ex`, but adding `:safe` in `utils.ex` does **not** by itself bound decompression, so both the size cap and `:safe` are worth applying.

## Did I verify this?
- **Verified by reading the neighbour module (`sync/pubsub.ex`, full):** the loopback/no-auth TCP transport, the frame format, decode-before-hash-check ordering, the unbounded 64-bit size field, and that `decode_data/1` calls the deserializer with default (`[]`) opts. The first-pass "outside this file" hedge on reachability is resolved — reachability is now source-confirmed, not assumed.
- **Verified by execution (this pass):** default `binary_to_term/2` interns a brand-new atom (`delta=1`); `[safe]` rejects an unknown atom (`{error,badarg}`). Prior-pass measurements (50 000 atoms/868 KB message; complete fun rejection) retained.
- **Not run / asserted:** the final atom-table-full VM abort (would crash the sandbox) — inferred from documented OTP behavior. The rate, `:safe` rejection, and fun rejection are measured.
- **Could not verify (upgraded to low-confidence finding):** whether an external tree-printer caller forwards an attacker-controlled `name`/`info` — the caller modules are not in this file or in the neighbour set available to me, so `CSCAN-utils-tree-printer-terminal-escape` remains low-confidence rather than dropped.

### Dropped in re-review
None. Every prior finding and ruled-out entry is retained; the sole prior caveat that hedged on unverifiable external callers was upgraded to a low-confidence finding rather than dropped.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`read_httpc` TLS retries (`utils.ex:908-921`)** — the ipfamily-fallback and `middlebox_comp_mode: false` retries both keep `verify: :verify_peer` + hostname check; `middlebox_comp_mode` only toggles the TLS 1.3 dummy-CCS, not certificate verification. No downgrade.
