# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/infer.exs
<!-- sha256: ef90989f937ef462551197313e25aa898bcf06ff711ad609ac3e334bdde65054 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/infer.exs
<!-- sha256: ef90989f937ef462551197313e25aa898bcf06ff711ad609ac3e334bdde65054 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

Re-review notes on the two flagged hedges (`reasoned`, `not`):
- **Producer trust ("...not on reading the Makefile")** — verified the *producer* half by reading the neighbour `src/elixir_erl.erl:147-148`: `checker_chunk(Contents, ChunkOpts) -> [{<<"ExCk">>, term_to_binary({checker_version(), Contents}, ChunkOpts)}]`. The consumer pattern `{_, checker} = :erlang.binary_to_term(checker_blob)` (line 43) destructures exactly the `{checker_version(), Contents}` 2-tuple this producer emits — round-trip confirmed. The invocation-site (Makefile) half is unresolved because the extracted tree contains no Makefile; re-ran the search (`infer.exs|scripts/infer|infer_signatures` across the tree → no matches). That is a scoping fact about the extracted tree, not an assumption I can clear by reading, so it is stated as such, not hedged.
- **`[:safe]`-breaks-the-build caveat ("reasoned, not empirically confirmed")** — now empirically tested (OTP 29 / ERTS 17.0.2 in the sandbox). See the finding's Evidence field.

## 1. Summary
A build-only script that re-infers stdlib types by rewriting Elixir's own freshly-compiled `.beam` files. Its single notable sink (`binary_to_term` without `:safe`, line 43) consumes the `ExCk` chunk that Elixir's own compiler produced moments earlier in the same build (`elixir_erl.erl:148`, now read and confirmed). No untrusted input reaches it under the documented invocation. One low-confidence defense-in-depth finding is retained.

## 2. Security

#### `CSCAN-infer-binary-to-term-unsafe`
- **Finding ID:** `CSCAN-infer-binary-to-term-unsafe`
- **Severity:** Low
- **Confidence:** low
- **Location:** `lib/elixir/scripts/infer.exs:43`, inside the `Task.async_stream` worker fn (`{_, checker} = :erlang.binary_to_term(checker_blob)`).
- **Class:** Deserialisation (CWE-502)
- **Issue:** `checker_blob` — the `ExCk` chunk read out of a `.beam` file (`File.read!` at line 33 → `:beam_lib.chunks/2` at lines 35-36) — is passed to `:erlang.binary_to_term/1` **without the `[:safe]` option**. Unsafe `binary_to_term` materialises arbitrary atoms (atom-table growth, CWE-770), decodes fun/pid/ref terms, and, if the blob is a `:compressed` term, can expand a small input into a large one (decompression-bomb memory blowup).
- **Impact / threat:** To influence `checker_blob` an attacker must control the `ExCk` chunk of a `.beam` file sitting in `lib/elixir/ebin`. Under the script's documented use that directory is Elixir's own stdlib output, and the chunk is written by Elixir's own compiler (`elixir_erl.erl:147-148`, `term_to_binary({checker_version(), Contents}, ChunkOpts)`) earlier in the same build — a trusted producer→consumer round-trip, not a dependency, archive, git ref, or externally-fetched file. Anyone able to plant a hostile `.beam` in that ebin already has write access to the build tree, and therefore arbitrary code execution when those modules load, so this sink grants nothing beyond what such an attacker already holds. It is additionally *not* a code-execution path: the decoded `checker` is only used as a data map (`update_in(checker.exports, …)` at lines 53-58 → `:elixir_erl.checker_chunk/2` at line 60), never `apply`'d. Defense-in-depth only; no reachable trust boundary.
- **Evidence:** verified (producer + consumer-usage read directly); the `:safe` fix-caveat is now empirically grounded. The sink and its unsafe form are read from `infer.exs:43`. The trusted-producer claim is confirmed by reading `src/elixir_erl.erl:147-148` (the `ExCk` chunk is `term_to_binary({checker_version(), Contents}, ChunkOpts)`), which is exactly the shape the consumer destructures. The decoded term's data-only usage is confirmed from `infer.exs:53-60`. No path feeds attacker-controlled `.beam` files to this script (re-confirmed: no Makefile/invocation site in the extracted tree).
- **Suggested fix:** Prefer `:erlang.binary_to_term(checker_blob, [:safe])` to bound the decoded term. **Empirically confirmed caveat (ran it, OTP 29 / ERTS 17.0.2):** I encoded a term containing a novel atom in one VM, wrote it to disk, and decoded it in a fresh VM where `list_to_existing_atom/1` on that name raised `badarg` (proving the atom was absent from the table). `binary_to_term(Blob, [:safe])` then raised `badarg`, whereas the unsafe form decoded it and created the atom. So `[:safe]` **will** hard-fail on any chunk that references an atom not already in the running build's atom table. Whether that breaks *this* build depends on whether every atom the `ExCk` term references is pre-loaded at decode time — the modules being processed come from `Application.spec(:elixir, :modules)` and are loaded (so their own function/type atoms exist), but a checker term may reference atoms from not-yet-loaded modules' remote types. I did **not** try `[:safe]` against a real stdlib `ExCk` chunk (none available in the sandbox), so the build-impact is unresolved; validate against a real chunk before merging. If `:safe` proves incompatible, this remains an accepted, documented build-internal round-trip.

### Ruled out
- **`File.read!` (line 33) / `File.write!` (line 76) path handling** — the path is `Path.join(ebin, "#{module}.beam")` where `ebin` is `Path.expand("../ebin", __DIR__)` (fixed location relative to the script) and `module` is an atom drawn from `Application.spec(:elixir, :modules)` filtered to `"Elixir." <> _`. No component comes from network/dependency/user input, and stdlib module atoms stringify with dots (`Elixir.URI`), yielding no `/` or `..`. No traversal reachable. Basis: lines 7, 14-15, 30.
- **Path traversal via `#{module}` interpolation into the filename** — same as above; `module` is a compiler-supplied atom for Elixir's own modules, not external bytes. Basis: lines 14-17, 30.
- **Concurrency / TOCTOU on the `.beam` files (`Task.async_stream` + write-back)** — reads happen inside the stream (line 33) but all `File.write!` calls are deferred until after `Enum.to_list()` has drained the stream (comment at lines 72-73, code at lines 74-78), so no `.beam` is rewritten while any worker is still reading/inferring. No interleaving that changes a file mid-inference. Basis: lines 28-78.
- **Atom creation from external strings (`String.to_atom`/`Module.concat`)** — none present; the only atom↔string conversion is `Atom.to_string/1` (line 15), the safe direction. Basis: full-file read.
- **Code execution / eval / command / network / EEx / regex-from-input** — no `Code.eval*`, `System.cmd`, `:os.cmd`, `EEx`, `Regex.compile`, or outbound request appears anywhere in the file. Basis: full-file read.
- **`:beam_lib.chunks/all_chunks/build_module` parsing crashes** — a malformed `.beam` makes the `{:ok, …} = …` / `{:debug_info_v1, …} = …` matches fail and aborts the build task; this is build robustness on Elixir's own artifacts, not a security boundary (and line 61 pins `^module`, an integrity check that rejects a beam whose internal module name disagrees with its filename). Basis: lines 35-38, 60-66.

## 3. Did I verify this?
- **Producer round-trip — VERIFIED (neighbour read).** Read `src/elixir_erl.erl:138-148`; `checker_chunk/2` emits `term_to_binary({checker_version(), Contents}, ChunkOpts)`, matching the consumer's `{_, checker}` destructure at `infer.exs:43`. First-pass hedge ("...not on reading the Makefile") is resolved for the producer half.
- **Decoded-term usage — VERIFIED (file read).** `checker` is consumed only as data (`update_in(checker.exports, …)` at lines 53-58, then `:elixir_erl.checker_chunk/2` at line 60); never applied/invoked. No code-execution path from the deserialised term.
- **`[:safe]` semantics — VERIFIED (executed).** Cross-VM test on OTP 29 / ERTS 17.0.2: `binary_to_term/2` with `[safe]` raises `badarg` on an atom absent from the atom table (confirmed absent via `list_to_existing_atom` raising first), while the unsafe form creates it. The first-pass "reasoned, not empirically confirmed" hedge is upgraded to executed evidence for the *mechanism*.
- **Build-impact of the fix — NOT verified.** I did not run `[:safe]` against a real stdlib `ExCk` chunk (none present in the sandbox), so whether `[:safe]` actually rejects a legitimate chunk in this build is unresolved. Stated as a caveat on the suggested fix, not as a settled claim.
- **Invocation site — NOT present.** Re-ran the search for `infer.exs` / `scripts/infer` / `infer_signatures` across the tree: no matches, no Makefile in the extracted tree. The "trusted producer" conclusion rests on the hardcoded `ebin`/module-spec inputs (lines 7, 14) and the confirmed `elixir_erl.erl` producer, not on the (absent) build-command file.
- **Script not executed end-to-end.** It requires a fully built stdlib with `ExCk` chunks and the `Module.Types` inference machinery; that is orthogonal to the sink analysis above.

### Dropped in re-review
None. Every ruled-out entry and the single finding from the first pass is preserved; the finding's `:safe` caveat was strengthened from reasoned to empirically-grounded rather than dropped or upgraded in severity.
