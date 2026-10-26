# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/file/stream.ex
<!-- sha256: 1d6b82eb01621a93d88f863d2d851f8dc995063446c081896f433a356898d821 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:46:21Z_

I have enough to conclude. I traced every sink in the file, confirmed the public-API contract in `file.ex`, and verified the edge-case behaviors by running them against the built-from-source Elixir. Here is the report.

---

## Summary

`File.Stream` is a thin streaming wrapper over `:file`/`:erpc` whose only external inputs (path, modes, file content) are caller-chosen or governed by the distributed-Erlang trust model; no untrusted-data-to-dangerous-sink path is reachable through its documented public API.

## Security

No findings.

This module exposes only the `File.Stream` struct plus its `Enumerable`/`Collectable` implementations (the `__build__/3` and `__open__/2` helpers are `@doc false`). Reached through the documented API (`File.stream!/3` → `Enum.*`/`Stream.*`/`Enum.into`), every sink resolves to one of: a caller-supplied path handed to `:file.open`/`:file_io_server:start`, a caller-chosen chunk size, or the local `node()`. None crosses a trust boundary this primitive is responsible for defending. I walked the Phase-1 sink classes (code eval, atom creation, command exec, deserialization, template/interpolation, crypto, atoms/dispatch, shared state) and found no instances; the only classes with live call sites are file/path handling, network (`:erpc`), and resource/consumption, all analyzed and ruled out below.

### Ruled out

- **Unconfined path to `:file.open` / `:file_io_server:start` / `File.stat`** (`__open__/2:57,62`; `Collectable.into/1:69`; `reduce/3:105`; `count/1:135`; `open!/3:164`). The `path` originates from the `File.stream!/3` caller (`IO.chardata_to_string(path)` in `file.ex:2265`); there is no `..`/root confinement, and there should not be — a low-level streaming primitive opens exactly the path it is given, and confinement (`Path.safe_relative/1`) is the caller's responsibility. No trust boundary is crossed inside this file. *Basis: verified by tracing the call chain in `file.ex:2263-2265`.*

- **`:erpc.call` to the struct's `node` field** (`__open__/2:62`, `count/1:135`). `File.stream!/3` always sets `node: node()` (`stream.ex:52`), so the remote branch only fires for a struct explicitly created on / forwarded from another node — the documented cross-node design (`file.ex:2202-2205`). This lives entirely within the distributed-Erlang trust model: any node sharing the cluster cookie already has full mutual code execution (`:rpc.call(node, :file, :read_file, …)`), so routing a file open through `:erpc` grants an attacker no capability the cookie boundary doesn't already grant. Not a vulnerability this file introduces. *Basis: verified `node: node()` construction; distributed-Erlang cookie is the pre-existing boundary.*

- **`div`/`rem` by zero and negative `line_or_bytes` in `count/1`** (`stream.ex:143-144`). `Enum.count` on a byte stream computes `rem(size, bytes)`/`div(size, bytes)`. With `bytes = 0` this raises `ArithmeticError`; with a negative it returns a nonsense count. Reaching it requires `File.stream!(path, 0)` / negative, which violates the documented `@spec … :line | pos_integer` (`file.ex:2250`) — the runtime guard `is_integer/1` (`file.ex:2188`) is merely looser than the spec. `line_or_bytes` is a programmer-selected chunk size, essentially never sourced from untrusted input, so this is an out-of-scope contract violation, not an attack path. *Basis: verified — `Enum.count(File.stream!(f, 0))` → `{:rescued, "bad argument in arithmetic expression"}`; `bytes: -5` → count `-3`.*

- **BOM detection quirk / dead 4-byte clause** (`bom_length/1:220-225`). `<<254, 255, _::binary>>` (UTF-16BE, line 221) shadows the later `<<254, 255, 0, 0, _::binary>>` clause (line 224, never reachable), and a real UTF-32LE BOM `FF FE 00 00` is matched as UTF-16LE and only 2 of 4 bytes are trimmed. This is a text-decoding correctness bug in `:trim_bom`, not a security issue: mis-trimming leading bytes reaches no security sink and crosses no trust boundary (the bidi/Trojan-Source and script-mixing defenses live in the tokenizer, not here). *Basis: verified — with `:trim_bom`, a `FF FE 00 00`-prefixed file streams back as `<<0, 0, "REST">>` (leading `00 00` not trimmed).*

- **TOCTOU between `File.stat` size and the BOM read in byte-`count/1`** (`stream.ex:135` vs `140`). The size is stat'd, then the file is separately opened to measure the BOM; concurrent modification yields an approximate count. This is inherent to counting by stat and is a correctness caveat only — a caller must not base a security decision on a streamed count. *Basis: reasoned from the two-step stat-then-open sequence.*

- **`:ok = :file.close(device)` MatchError on `delayed_write`** (`Collectable.into/2:89,95`). A deliberately-documented crash (comments at lines 87-88, 93-94): a failed delayed write surfaces as a `MatchError` in the caller's own process. It affects only the caller, carries no attacker-controlled effect, and is intended behavior. *Basis: confirmed by the in-code comments.*

## Risks / caveats

- I did not exercise the genuine multi-node `:erpc` path (single-node sandbox); the dismissal rests on the distributed-Erlang cookie already being a full-trust boundary, and on `File.stream!/3` hard-coding `node: node()`, both verified by source. A caller who hand-builds a `%File.Stream{node: …}` from untrusted input is outside the documented API and outside this module's contract.
- Findings were scoped to the documented public surface (`File.stream!/3` + `Enumerable`/`Collectable`). Behavior of the downstream `IO.each_binstream/2`, `IO.each_stream/2`, and `normalize_modes/2` (all in other modules) was treated as out of scope; e.g. unbounded memory when streaming a newline-free file by `:line` is a property of `IO.each_binstream`, not of this file.
- The BOM/UTF-32 mis-trim and the loose `is_integer` guard are real correctness defects worth a maintainer's attention, but neither is security-relevant under the stated threat model, so they are recorded as ruled-out rather than findings.

