# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/diff.exs
<!-- sha256: 818c4831769f25e726031e89c90a3464867ce9906c8ac4366810a7889795d85e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T13:00:14Z_

I have everything I need, verified end-to-end. Let me finalize the analysis.

**Confirmed facts (all `verified` by running against the built OTP-29 / Elixir 1.21.0-dev toolchain):**
- Line 107 feeds raw `Docs`/`ExCk` BEAM-chunk bytes to `:erlang.binary_to_term/1` with **no `:safe`** and no size cap.
- `:beam_lib.chunks/2` returns those chunks **raw**; real Elixir beams carry both, so the `@term_chunks` path is naturally reached (and I proved the `@atom_chunks` path does not short-circuit when only `Docs` differs).
- End-to-end through `Diff.compare_files/2`, two crafted `.beam` files minted **40,115 atoms** in a fresh VM (`:safe` refuses them; cap is 1,048,576 → crash at scale).
- The compressed-term format (observed tag `<<131,80,…>>`) gives **1028×** inflation with no size bound, then rendered by `inspect(limit: :infinity)`.
- `System.cmd` paths through `compare_dirs` are absolute → no option injection.

---

# Security review — `elixir/lib/elixir/scripts/diff.exs`

## Summary
A build-artifact diff script that unsafely decodes attacker-controllable BEAM-file chunk bytes with `:erlang.binary_to_term/1` (no `:safe`, no size bound), enabling atom-table exhaustion and memory/CPU denial-of-service of the diffing process; no code-execution or path-traversal issues were found.

## Security

#### `CSCAN-diff-unsafe-binary-to-term`
- **Finding ID:** `CSCAN-diff-unsafe-binary-to-term`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `diff.exs:107` (formatter in `beam_diff/4`), decoded in `chunk_diff/4` at `diff.exs:138-139`
- **Class:** Deserialisation
- **Issue:** `beam_diff/4` builds the `@term_chunks` (`ExCk`, `Docs`) formatter as `&inspect_all(:erlang.binary_to_term(&1))`. `:beam_lib.chunks/2` returns those chunks as their **raw** on-disk bytes, and `chunk_diff/4` passes them straight into `:erlang.binary_to_term/1` with no `:safe` option. Those bytes are entirely controlled by whoever produced the `.beam` file being compared. Unsafe `binary_to_term` mints arbitrary atoms; the atom table is never garbage-collected and is globally capped (1,048,576 by default), so a crafted `Docs`/`ExCk` chunk carrying a large list of distinct atoms exhausts it and crashes the VM (CWE-502 / CWE-770).
- **Impact / threat:** An attacker who controls a `.beam` file in one of the compared directories — e.g. a malicious dependency's compiled artifact, or any untrusted build output fed to `Diff.compare_dirs/2` (the CLI entry) or `Diff.compare_files/2` — crashes the diff process by atom-table exhaustion. The trust boundary crossed is untrusted file/artifact bytes on disk reaching an unsafe deserialiser. Impact ceiling is DoS of a short-lived dev/CI process (the decoded term is only `inspect`ed, never applied, so no code execution); reach is limited to this reproducibility-check script rather than the default `mix` build path.
- **Evidence:** `verified`.
  - `:safe` vs unsafe on a fresh VM: `:erlang.binary_to_term(payload, [:safe])` → `{:refused, ArgumentError}`; `:erlang.binary_to_term(payload)` → mints the atom (`is_atom=true`).
  - End-to-end through the file's own module: two tampered beams (differing only in `Docs`) run through `Diff.compare_files("/tmp/f1.beam","/tmp/f2.beam")` in a fresh VM:
    ```
    atom_count before: 20581
    atom_count after : 60696
    NEW ATOMS MINTED from attacker-controlled .beam bytes: 40115
    ```
    with `:erlang.system_info(:atom_limit)` = `1048576` (scaling the crafted list toward that cap crashes the VM).
- **Suggested fix:** Decode with the safe option and reject failures: `:erlang.binary_to_term(&1, [:safe])`. Because `:safe` bounds *type* but not *allocation* (see next finding), also bound chunk size before decoding.

#### `CSCAN-diff-beam-chunk-decode-bomb`
- **Finding ID:** `CSCAN-diff-beam-chunk-decode-bomb`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `diff.exs:107` (decode) + `diff.exs:101-103` `inspect_all/1` (unbounded render)
- **Class:** Resource consumption
- **Issue:** The `Docs`/`ExCk` chunk is a compressed external-term-format payload (real chunks begin `<<131, 80, …>>`, tag `80` = COMPRESSED). `:erlang.binary_to_term/1` transparently decompresses with no size limit, so a tiny crafted chunk inflates to a huge in-memory term, which is then handed to `inspect_all/1` = `inspect(data, pretty: true, limit: :infinity)` — `:infinity` disables truncation — and the resulting giant string is written to a temp file and run through `diff`. Note `[:safe]` does **not** mitigate this; a separate size bound is required.
- **Impact / threat:** A single small malicious `.beam` (tens of KB) causes hundreds of MB to GB of heap allocation plus O(size) string construction and temp-file I/O, hanging or OOM-killing the diff process. Same trust boundary (untrusted artifact bytes) and same limited reach as the finding above, but an independent defect that survives the `:safe` fix.
- **Evidence:** `verified`. A `term_to_binary(:binary.copy(<<0>>, 80_000_000), [:compressed])` payload is `77780` bytes on disk and inflates via `:erlang.binary_to_term/1` to `80000000` bytes — **1028× amplification** — exactly the primitive at line 107; `inspect(limit: :infinity)` then renders the whole inflated term without truncation.
- **Suggested fix:** Reject oversized chunks before decoding (e.g. cap `byte_size(chunk)` and/or decode under a bounded worker), and use a finite `limit:` for the diagnostic `inspect` instead of `:infinity`.

### Ruled out
- **`System.cmd("diff"/"xxd", …)` option injection (`diff.exs:156,161`)** — `verified`. `System.cmd/3` uses explicit argv (no shell), so no command injection. Through the CLI/`compare_dirs/2` every path handed to `diff`/`xxd` is built by `Path.join(Path.expand(dir), rel)` and is absolute: `Path.join(Path.expand("d"), Path.relative_to("/abs/-x.beam","/abs"))` → `"/work/d/-x.beam"`, so a `-`-leading filename can never reach the tools as an option operand. `compare_files/2` reads paths chosen by its caller, not attacker-forwarded through this file.
- **`Path.wildcard(dir <> "/**")` cost / symlink following (`diff.exs:167`)** — the two roots are chosen by the operator running the script, not attacker-supplied; not a crossed trust boundary.
- **`write_tmp/1` fixed `./tmp` dir with predictable-ish names (`diff.exs:178-183`)** — `System.unique_integer([:positive])` is unique per run and the write target is the operator's own CWD; exploiting it needs a pre-existing local attacker with write access to CWD, which is not a boundary this script defends.
- **Code-execution via the decoded term** — the `binary_to_term` result flows only to `inspect` → temp file → `diff`; it is never `apply`ed, `eval`ed, or called, so the deserialisation does not yield RCE (DoS only).

## Risks / caveats
- **Reach is narrow and impact is DoS-only.** This is an internal reproducibility-check script (`scripts/diff.exs`), not on the `mix deps.get`/`compile`/`archive.install` path; the attacker must get a crafted `.beam` into a directory the operator deliberately diffs. I rated both findings Medium on that basis — a reviewer prioritising only default-build-path exposure could reasonably treat them as Low.
- **Upstream `beam_lib` decoding not in scope.** The `@atom_chunks` path (`debug_info`, `attributes`, `compile_info`, …) makes `:beam_lib.chunks/2` itself decode chunk contents (e.g. `Dbgi` holds a `term_to_binary` payload) inside OTP. Whether that OTP-side decoding is equally unbounded on hostile input is a property of `beam_lib`, not this file, so I did not adjudicate it; if the concern is untrusted `.beam` handling generally, that path deserves its own look.
- I could not exercise the `@binary_chunks`/final xxd fallback end-to-end because `xxd` is absent in this sandbox, but those paths do not add a deserialisation or injection sink beyond what is analysed above (they shell out to `xxd`/`diff` on absolute or `tmp/`-prefixed paths).

