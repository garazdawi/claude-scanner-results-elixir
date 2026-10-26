# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/cover.exs
<!-- sha256: 410fbc7779d6b2b2866d9015acd8ffa3cce7413f89c7a21c6f3e034ba73d2de9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:50:48Z_

I now have a complete picture. Let me confirm the deserialization mechanism and trust boundaries are fully characterized.

**Verified facts:**
- `:cover.import/1` (OTP `tools-4.2.1/src/cover.erl:2994-2997`) deserializes via `binary_to_term/1` with `{nowarn_possibly_unsafe_function, ...}` — i.e., **not** `:safe`.
- `cover_dir()` resolves to `Path.join(__DIR__, "../../..")` + `"cover"` — a **fixed compile-time path**, not influenced by `System.argv` or any env var (the `COVER` env var only gates `enabled?/0`, which `cover.exs` never calls).
- The glob pattern `ex_unit_*.coverdata` is a fixed literal; `String.to_charlist/1` on the path creates no atoms.

Here is the report.

---

# Security review — `lib/elixir/scripts/cover.exs`

## Summary
This is an internal build-time coverage-combining script whose only inputs are fixed, compile-time-derived local paths and the project's own `.coverdata` build artifacts; no untrusted trust boundary is crossed, so there are no actionable findings.

## Security

`No findings`

### Ruled out

- **`:cover.import` deserializes `.coverdata` via unsafe `binary_to_term` (line 15).** *Basis: verified.* `:cover.import/1` reads coverdata with `binary_to_term/1` and no `:safe` (`/usr/local/lib/erlang/lib/tools-4.2.1/src/cover.erl:2994,2997`, with `{nowarn_possibly_unsafe_function,{erlang,binary_to_term,1}}` at line 109), so a hostile `.coverdata` file could cause atom-table exhaustion / large-allocation DoS (no code exec from `binary_to_term` alone). However, the inputs come from `CoverageRecorder.cover_dir()` = `Path.join(Path.join(__DIR__, "../../.."), "cover")` matched against the fixed literal glob `ex_unit_*.coverdata` — a fixed local directory inside the build tree, verified to be uninfluenced by `System.argv`/env. These files are the project's *own* `:cover.export` output from its `mix test` run, not a dependency, archive, git ref, or externally-supplied file. Planting a malicious `ex_unit_*.coverdata` requires write access to the build tree, which already grants code execution. The vulnerable decoder is OTP's, and `:cover.import` exposes no `:safe` option, so there is no in-file fix. Trust boundary not crossed → not a finding for this file.
- **`Code.require_file("cover_record.exs", __DIR__)` — arbitrary code execution at load (line 6).** *Basis: verified (by reading).* The path is the fixed string `"cover_record.exs"` resolved against `__DIR__` (the script's own directory); no external component. Controlling that file means already controlling the source tree. Not a finding.
- **`Path.wildcard` / `Path.join` path handling (lines 10, 12, 28).** *Basis: verified (by reading).* Every path component is a literal or the fixed `cover_dir()`; no external component reaches a path, so no traversal or pattern injection. `String.to_charlist/1` (lines 15, 18) converts paths to charlists and creates no atoms.
- **`Mix.Tasks.Test.Coverage.generate_cover_results` output writing (lines 26-30).** *Basis: verified (by reading).* `output:` is the fixed `cover_dir()`; the module names it reports come from locally compiled beams (trusted). No external input steers the write location.
- **`Process.group_leader/2` + `StringIO.open("")` (lines 23-24).** No external input; purely redirects cover's log output. Not security-relevant.

## Risks / caveats
- **Scope:** I reviewed only `cover.exs`. It transitively invokes `CoverageRecorder` (line 7), whose `skip_from_coverage?/1` calls `String.to_unsafe_atom/1` on `.beam` basenames (`cover_record.exs:88`) — an atom-creation sink — but the defective code lives in `cover_record.exs` (out of scope) and is fed by the local `lib/*/ebin/*.beam` build output (trust boundary not crossed).
- **Residual risk I could not fully eliminate:** In a CI topology where `ex_unit_*.coverdata` artifacts are cached/restored across jobs or produced by an untrusted PR's test run and later combined by this script, a poisoned `.coverdata` would reach `:cover.import`'s unsafe `binary_to_term` (verified unsafe above), yielding at most an atom/memory DoS on the build host. This depends on out-of-file CI wiring not visible here; under this file's own threat model (fixed local build artifacts) it does not qualify. If such a pipeline exists, the mitigation belongs upstream (in OTP `:cover` gaining a safe-import mode, or in the CI's artifact-integrity controls), not in this script.
- I did not execute a full `COVER=1 mix test` + `cover.exs` run end-to-end; the trust-boundary conclusions rest on reading this file plus `cover_record.exs` and confirming the OTP decoder mechanism and the absence of argv/env influence on the paths, all of which were verified by inspection.

