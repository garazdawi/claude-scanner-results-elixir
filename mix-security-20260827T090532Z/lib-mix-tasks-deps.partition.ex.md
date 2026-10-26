# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.partition.ex
<!-- sha256: 7eec7a7f57151c75e34351f76531428f4f872502d8fc6b20e48ea2ca51583fb6 -->

_Analysed on 2026-08-27T10:56:15Z_

All facts verified. Cross-VM test confirms `:safe` rejects novel atoms (`:rejected_by_safe`) while the bare `binary_to_term` creates them. I now have everything needed.

Summary of what I verified by running against the built-from-source toolchain (Elixir 1.21.0-dev / OTP 29):
- `String.to_unsafe_atom/1` creates fresh atoms with no existence check.
- The coordination socket (`:gen_tcp.listen(0, ...)`, line 11) binds to `0.0.0.0` (all interfaces), while clients connect to `127.0.0.1`.
- `File.mkdir_p` accepts a pre-existing symlinked directory and `File.write` follows it — an attacker who squats `/tmp/mix_deps_partition` sees the random filename and can overwrite it.
- `binary_to_term` without `:safe` deserializes attacker-controlled `code_path`/`compile_config`; `:safe` would at least reject novel atoms.
- Server-loop parsing crashes on malformed TCP input (`MatchError`, `ArgumentError`, `KeyError`), and a known-app + novel-status loop creates unbounded atoms.

## Summary

`Mix.Tasks.Deps.Partition` (the parallel dependency-compiler enabled by the documented `MIX_OS_DEPS_COMPILE_PARTITION_COUNT`) writes its config to a predictable world-shared temp path and deserializes it with an unsafe `binary_to_term`, and coordinates workers over an unauthenticated TCP socket bound to all interfaces that feeds untrusted line data into atom creation and non-total pattern matches.

## Security

#### `CSCAN-deps.partition-config-tmpfile-hijack`
- **Finding ID** — `CSCAN-deps.partition-config-tmpfile-hijack`
- **Severity** — Medium
- **Confidence** — medium
- **Location** — `deps.partition.ex:43-46` (`server/4`, temp-dir write) and `deps.partition.ex:264-271` (`client/0`, read + deserialize)
- **Class** — File operations and path handling (predictable temp file / squatting + TOCTOU) composed with Deserialisation (`:erlang.binary_to_term/1` without `:safe`)
- **Impact** — On a shared/multi-user build host (e.g. a CI runner), a local unprivileged user can pre-create `/tmp/mix_deps_partition` (as a directory they own, or a symlink to one). When a victim runs `mix deps.compile` with `MIX_OS_DEPS_COMPILE_PARTITION_COUNT>1`, the server writes the serialized config there under a random name; the attacker gets the filename instantly (their directory) and overwrites it during the spawned worker's VM-boot window before `File.read!` (line 268). The client then `binary_to_term`s the attacker's payload and uses its fields directly: `:code.set_path(code_path)` (line 274), `Application.put_all_env(compile_config, ...)` (line 279), and `Mix.ProjectStack.push(peek.name, peek.config, peek.file)` → `Mix.CLI.main(...)` (lines 288-289). Attacker-controlled `code_path` / project config yields code execution as the victim during the build. Absent the overwrite, squatting still leaks the serialized project config into the attacker's directory, and if the victim's write fails the code falls back to putting the config on the process argv (visible via `ps`).
- **Evidence** — `verified`. `File.mkdir_p` on a pre-existing symlink→dir returned `:ok`; `File.write` followed the symlink (`write via symlinked dir: :ok`); the attacker read the file back and overwrote it (`what client would read back: "ATTACKER_PAYLOAD"`). Round-tripping a crafted map through `Base.url_encode64 |> Base.url_decode64! |> binary_to_term` reproduced attacker-controlled `code_path` (`[~c"/home/attacker/evil_ebin"]`) and `compile_config`. Cross-VM test confirmed `binary_to_term(bin, [:safe])` rejects a novel atom (`:rejected_by_safe`) whereas the bare call (used here) accepts and creates it.
- **Suggested fix** — Do not place the config under a fixed, world-shared directory name. Create a per-invocation directory with `File.mkdir_p!` of a random component and mode `0o700` (or use an OS-secure temp facility), verify it is not a symlink / is owned by the caller before writing, and pass the payload out-of-band rather than through a predictable path. Add `[:safe]` to the `:erlang.binary_to_term/1` call (line 271) as defense-in-depth — but note `:safe` alone is insufficient because the *content* (`code_path`, project config) is still trusted; the source's integrity is the real fix.

#### `CSCAN-deps.partition-coord-socket-atom-dos`
- **Finding ID** — `CSCAN-deps.partition-coord-socket-atom-dos`
- **Severity** — Medium
- **Confidence** — medium
- **Location** — `deps.partition.ex:11` (`server/3`, `:gen_tcp.listen`), `deps.partition.ex:99-103` (accept loop), `deps.partition.ex:154-159` (`server_loop/4`)
- **Class** — Network (unauthenticated socket bound to all interfaces) composed with Atom creation (CWE-770) and Resource consumption / DoS
- **Impact** — The coordination listener is opened with `:gen_tcp.listen(0, ...)` and no `ip:` option, so it binds `0.0.0.0` (all interfaces) though workers only ever dial `127.0.0.1`. There is no authentication: any peer that connects during the compile window is treated as a worker. A co-tenant local user (or, because of the all-interfaces bind, an off-host attacker who reaches the ephemeral port) can (1) crash the build by sending a non-integer index (`String.to_integer`, line 102 → `ArgumentError`) or an out-of-range index (`Map.fetch!`, line 103 → `KeyError`); (2) once holding a slot, crash `server_loop` by sending data without exactly one `:` (`[app, compiled?] = ...`, line 155 → `MatchError`) or an unknown app (`Map.replace!`, line 159 → `KeyError`); or (3) exhaust the atom table by repeatedly sending `<known-dep>:<novel>` lines — each `String.to_unsafe_atom` (line 156) mints a fresh atom with no existence check and no crash, eventually aborting the BEAM. All of this runs on the build path, where a crash aborts the developer's compile.
- **Evidence** — `verified`. `:inet.sockname` on the exact listen options returned `{{0,0,0,0}, <port>}` (“BOUND TO 0.0.0.0 (ALL INTERFACES)”). The parse reproductions returned `MatchError` for 1-field and 3-field inputs, `ArgumentError` for a non-integer index, and `KeyError` for an unknown app in `Map.replace!`; the known-app + novel-status loop ran without crashing, and `String.to_unsafe_atom` was shown to create fresh atoms unconditionally.
- **Suggested fix** — Bind the listener to loopback (`:gen_tcp.listen(0, [..., ip: {127,0,0,1}])`) and, ideally, require the accepted client to present a per-run shared secret generated by the server before it is trusted. Replace `String.to_unsafe_atom` on socket data with `String.to_existing_atom` (dependency apps already exist as atoms) and make the message parsing total (match on a 2-element split, handle unknown apps) so malformed input cannot crash the coordinator.

### Ruled out
- **`Port.open({:spawn_executable, elixir}, args: ...)` (line 92)** — no shell is involved and args are a discrete argv list, so no command/shell injection; the `elixir` path comes from an operator env var / `System.find_executable`, i.e. operator-controlled, not attacker-controlled.
- **`run/1` connecting to `--host`/`--port` (lines 295-301)** and **client-side `String.to_unsafe_atom` (line 316)** — the host/port are supplied by the spawning server; an attacker can only influence them by running the task themselves (self-inflicted), and `client/0`'s `binary_to_term` is not reachable via `mix deps.partition` (only via the server's `-e` spawn or the temp-file hijack already covered above).
- **`IO.write([... data ...])` of child port output (line 178) / debug `IO.puts` (lines 82, 88)** — the bytes are this build's own child-compiler stdout; unescaped terminal sequences in compiler output are a general Mix property, not introduced here.
- **Fallback `{~c"--config", config}` placing the payload on argv (line 49)** — an information-exposure nuance (process-listing visibility of project config) folded into the first finding; not separately security-critical as the config holds no secrets by construction.

## Risks / caveats
- I verified every primitive (the `0.0.0.0` bind, symlink-following writes, TOCTOU filename disclosure, unsafe `binary_to_term`, and each crash/atom sink) in isolation on the built-from-source toolchain, but did **not** drive the full end-to-end spawn (`server/3` launching real `elixir` workers) to completion, so the exact width of the TOCTOU race window in Finding 1 and the precise number of atoms needed to abort the VM in Finding 2 are reasoned rather than measured.
- Both findings require the documented but non-default `MIX_OS_DEPS_COMPILE_PARTITION_COUNT>1` to be set and a project with ≥2 fetchable, not-yet-compiled deps; Finding 1 further requires a shared multi-user host, and Finding 2's off-host reach requires no host firewall on the ephemeral port (the local co-tenant DoS does not). A reviewer weighting these preconditions heavily could reasonably down-rate either to Low.
- I did not audit the downstream modules invoked from the deserialized payload (`Mix.ProjectStack.push`, `Mix.CLI.main`, `Mix.Tasks.Deps.Compile.compile_single`); the code-execution conclusion in Finding 1 rests on project config / `code_path` being build-controlling, which is standard Mix behavior, not on a specific sink I traced into those files.

