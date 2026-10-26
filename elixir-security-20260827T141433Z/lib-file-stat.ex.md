# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/file/stat.ex
<!-- sha256: fb58203093233e9fe04ccdeaf89610166d4e6cd95c1fceb4b5044d01bfb82508 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/file/stat.ex
<!-- sha256: fb58203093233e9fe04ccdeaf89610166d4e6cd95c1fceb4b5044d01bfb82508 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

I re-read the entire file, re-ran the round-trip / malformed-input reproductions, and read the one neighbour that consumes this module's output (`lib/file.ex`) to resolve the single hedge (`Basis: asserted`) in the first-pass report.

---

## Summary
`File.Stat` is a trivial, side-effect-free struct↔record mapping module (`to_record/1`, `from_record/1`) with a single compile-time record extraction from a fixed OTP header. No untrusted data reaches any dangerous sink in this file, and there are no security findings. The only prior hedge (the downstream-sink routing claim) is now verified against `file.ex` and its disposition is unchanged: the sink lives in another file, and this module neither adds privilege nor removes sanitization.

## Security

No findings.

The module's entire public surface is two pure functions plus a struct definition:

- `to_record/1` (`stat.ex:86`) pattern-matches a `%File.Stat{}` and rebuilds the fields, in order, into a `{:file_info, …}` tuple. No sink.
- `from_record/1` (`stat.ex:96`) pattern-matches a `{:file_info, …}` tuple and rebuilds a `%File.Stat{}`. No sink.
- The compile-time `Record.extract(:file_info, from_lib: "kernel/include/file.hrl")` (`stat.ex:59`) reads a **literal, fixed path** inside the trusted Erlang/OTP `kernel` application. The derived `keys`/`vals`/`pairs` feed the `unquote_splicing` macro expansions — all computed from the trusted record definition, never from caller input.

There is no code evaluation, no runtime atom creation, no command/OS interaction, no path construction from external components, no network, no crypto, no shared-state write, and no concurrency in this file.

### Ruled out

- **Deserialisation (`from_record/1`, stat.ex:96)** — *Basis: verified.* It does not decode external bytes; it consumes an **already-parsed** Erlang term and can only ever produce a `File.Stat` struct (no arbitrary-structure instantiation). Malformed/hostile tuples raise `FunctionClauseError` rather than decoding. Re-ran `File.Stat.from_record/1` on `{:file_info,1,2,3}` and `:whatever` → both raised `FunctionClauseError`. This is a documented-contract violation (the spec requires `:file.file_info()`), not an exploitable path.
  - *Axes examined:* arity/shape mismatch (short tuple), wrong tag (non-`:file_info` term). *Not examined:* none — the only structural inputs a caller can pass are a well-shaped `:file_info` tuple (accepted, pure remap) or anything else (raises); both tried.
- **Atom creation (CWE-770)** — *Basis: verified by reading the full file.* No `String.to_atom`/`binary_to_atom`/`Module.concat`/etc. The struct field atoms are fixed at compile time from the record; no runtime conversion of external bytes to atoms.
  - *Axes examined:* whole-file scan for atom-producing sinks. *Not examined:* none — the file is 100 lines and contains no runtime atom conversion.
- **Compile-time execution (`Record.extract`, stat.ex:59)** — *Basis: verified by reading.* The `from_lib:` argument is a compile-time string literal pointing into the trusted OTP install; no project/dependency/network/user-controlled input influences it, so the "build machine is a target" concern does not apply.
  - *Axes examined:* argument provenance (literal vs. computed). *Not examined:* none — the argument is a string literal on the same line.
- **Round-trip integrity (`to_record`/`from_record`)** — *Basis: verified.* Re-built a fully-populated `%File.Stat{}`, confirmed `to_record/1` yields `{:file_info, 10, :regular, :read_write, 1, 2, 3, 420, 1, 0, 0, 42, 1000, 1000}`, and `from_record(to_record(x)) == x` returned `true`. The two functions share the same compile-generated field order (both derive from `pairs`/`vals`), so no field-shuffling or meaning drift is possible.
  - *Axes examined:* field ordering and value preservation on a fully-populated struct (executed). *Not examined:* none material — both functions are generated from the same `keys`, so ordering cannot diverge between them by construction.
- **Downstream sink via `to_record` output** — *Basis: verified (routing); no finding.* **Upgraded from hedged assumption in first pass.** I read the consumer: `File.write_stat/3` at `file.ex:661-663` calls `:file.write_file_info(IO.chardata_to_string(path), File.Stat.to_record(stat), opts)`, confirming the record produced by `to_record/1` flows directly into `:file.write_file_info/2`. Setting `mode`/`uid`/`gid`/times from those values is that API's documented purpose, and the values are entirely caller-supplied (the caller constructs the `%File.Stat{}`). `to_record/1` performs a pure field remap — it neither adds privilege nor sanitizes, so there is nothing in *this* file to weaken. The sink itself resides in `file.ex`, out of scope for this file; no finding here.
  - *Axes examined:* the routing claim ("to_record output reaches `:file.write_file_info`"), now verified against `file.ex:663`. *Not examined:* none for this file's disposition — the sink is in `file.ex` and is that module's obligation to review.

## Risks / caveats
- Reproductions were re-run with `/opt/elixir/bin/elixir` (built from the source under review). Round-trip, record shape, and malformed-input raises all reproduced as reported.
- I judged only what this file owns. `from_record/1` returns whatever the OS `stat` syscall reports (an attacker can influence a file's own size/mtime/mode), but those become inert struct fields with no sink in this file; whether a *caller* makes a trust decision on them is that caller's obligation.
- The `from_lib` extraction is version-sensitive: if a future OTP `file_info` record gained or reordered fields, the generated clauses would still be internally consistent (both functions regenerate from the same `keys`/`vals`/`pairs`), so this is a correctness/compat concern at most, not a security one.

## Did I verify this?
- **File contents:** Yes — read all 100 lines.
- **Round-trip / malformed-input behavior:** Yes — executed against the built toolchain (see reproductions above).
- **Downstream routing of `to_record/1` output:** Yes — read `file.ex:661-663`; `to_record(stat)` is passed as the info argument to `:file.write_file_info/2` inside `write_stat/3`. The security disposition (no finding in this file) is unchanged.
- **Standing-context neighbour modules (`elixir_tokenizer`, `elixir_interpolation`, etc.):** Not applicable — that threat model covers the Erlang compiler front-end (`lib/elixir/src`), which is unrelated to this pure Elixir struct-mapping module; none of its Unicode/tokenizer obligations touch `stat.ex`.

### Dropped in re-review
None. Every prior ruled-out entry is retained; the sole hedged entry ("Downstream sink via `to_record` output", previously `Basis: asserted`) was upgraded to a verified routing claim with an unchanged no-finding disposition, not dropped.
