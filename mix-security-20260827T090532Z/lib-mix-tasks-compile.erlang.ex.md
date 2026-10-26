# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.erlang.ex
<!-- sha256: 1b25fbe7e940f23492b8421057c388dbcb90571b6e18955a3587d5c8b8163aba -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/compile.erlang.ex
<!-- sha256: 1b25fbe7e940f23492b8421057c388dbcb90571b6e18955a3587d5c8b8163aba -->

_Re-analysed on 2026-08-27_

Re-review to resolve the first-pass "single-file review" hedge. I read the delegated module `Mix.Compilers.Erlang` (`elixir/lib/mix/lib/mix/compilers/erlang.ex`) in full and re-ran the atom / wildcard axes on the OTP‑29 / Elixir toolchain in the sandbox. The delegated helpers introduce no new sink reachable from this file's inputs, and every prior ruled-out entry survives. Conclusion is unchanged: **No findings.**

## Summary
`Mix.Tasks.Compile.Erlang` scans and compiles `.erl` source files; all of its potentially-dangerous operations (atom creation from filenames, `:code.purge`/`:code.delete`, `:epp.parse_file`, `:compile.file`) are either bounded by real on-disk files or fall inside Mix's explicit "compiling code runs code" trust boundary, and the crash vectors I could construct are not reachable through the documented `mix compile.erlang` flow. The delegated `Mix.Compilers.Erlang.compile_entries/6` (previously unread) receives only filename-derived, `compile_path`-confined data from this file and adds no cross-boundary capability.

## Security

No findings

### Ruled out

- **`String.to_unsafe_atom/1` on dependency-controlled filenames (lines 94, 134, 251) — atom-table / crash DoS.**
  Confidence: high. The three atom conversions take a `.erl` basename a dependency/archive can influence. I re-ran the abuse axes on the sandbox toolchain (Elixir on OTP‑29): a 256-char input raises `SystemLimitError`, a 255-char input succeeds, and an invalid-UTF‑8 byte (`<<0xFF>>`) raises `ArgumentError` — i.e. `to_unsafe_atom` performs the same length/encoding validation as `to_atom`; "unsafe" only means "may intern a new atom." Both crash vectors are unreachable via the real entry path: (a) the over-length atom needs a single filename component >255 bytes, but `NAME_MAX` is 255 and `.erl` is stripped first, so the atom is ≤251 chars; (b) the invalid-UTF‑8 filename is filtered out by `Mix.Utils.extract_files` → `Path.wildcard`. I re-verified (b): under `+fnu`, `Path.wildcard("src/**/*.erl")` on a directory containing a real `src/mod\xFF.erl` returned only `["src/good.erl"]`. Atom-table exhaustion is bounded by the number of distinct real files on disk, and `String.to_existing_atom` is inapplicable because the module is being created by the very compilation about to run.
  Axes examined: input length (255/256), invalid UTF‑8 byte, wildcard filtering under `+fnu`, `Path.basename |> Path.rootname` slash-stripping (`"a/b/../c.erl"` → `"c"`). | Not examined: `+fnl`/`+fnu_w` and other native-encoding modes for the wildcard-surfacing-raw-bytes case (see caveat).

- **`:code.purge(module)` / `:code.delete(module)` in the compile callback (lines 94‑96) — VM-global code table manipulation from a crafted filename.**
  Confidence: high. A dependency shipping `src/lists.erl`/`src/code.erl` would purge/delete a live system module named after the file (the atom comes from the filename, not the `-module` attribute). Reaching the callback requires the file to be discovered, parsed successfully, and compiled — i.e. the dependency's Erlang is being compiled, which per Mix's documented threat model already grants compile-time code execution (parse transforms run inside `:compile.file`). A dependency that can trigger this can already `erlang:halt/0` or worse, so the purge adds no capability across the trust boundary. A benign filename collision (e.g. a project's own `src/json.erl`) is the intended "clear stale loaded module before recompiling" behaviour and is developer-self-inflicted.
  Axes examined: filename-derived (not `-module`-derived) atom source; trust-boundary reachability. | Not examined: none material — the trust-model objection makes exact VM failure mode (hang vs survivable) immaterial to the verdict.

- **Missing `{:exit, reason}` clause in `scan_sources` (lines 148‑151) and the equivalent in the delegated `compile_entries` parallel path (erlang.ex:157).**
  Confidence: high. `scan_sources`' `Enum.flat_map` only matches `{:ok, {:ok, _}}` / `{:ok, :error}`, and the delegated `compile_entries` parallel reducer only matches `{:ok, result}` (erlang.ex:157) — a crashed task would yield `{:exit, reason}` and abort the whole run with a `function_clause`/EXIT propagation instead of a clean error. Now that I have read `compile_entries`, I confirmed the only crash source in its parallel branch is the callback (this file's closure), which raises via `Mix.raise` only when `:compile.file` returns a **bare `:error`** — the documented "invalid `:erlc_options`" case, an operator-config fault, not attacker-reachable. A malicious dependency whose source fails to compile yields `{:error, errors, warnings}` (handled by `do_compile`'s clause, erlang.ex:289), never bare `:error`. In `scan_sources`, `module_from_artifact`'s atom crash is unreachable (above) and `:epp.parse_file` reports malformed input as `{:error, _}`/per-form error tokens (handled by `do_form`'s catch-all) rather than raising, and `tl(forms)` is safe because a successful parse always yields ≥1 form. Latent robustness gap, not attacker-triggerable.
  Axes examined: crash sources feeding each `Task.async_stream` reducer (both files now read); bare-`:error` vs `{:error,_,_}` return shapes of the callback. | Not examined: none material.

- **`:epp.parse_file/3` include handling (line 163) — arbitrary compile-time file read via `-include`.**
  Confidence: high. A scanned `.erl` can `-include("/etc/…")`, but EPP only tokenises included bytes into the form stream; `scan_source` inspects only `-file`/`-behaviour`/`-compile` attributes and never returns included content to any attacker, and non-Erlang content fails the parse and is skipped. No disclosure boundary crossed; the file-read is EPP behaviour inherent to compilation.
  Axes examined: which form attributes are retained (`do_form`), disclosure sink existence. | Not examined: none material.

- **Compile-time parse-transform execution (`add_parse_transforms`, `:compile.file`).**
  Confidence: high. Executing `{:parse_transform, _}` code and running the compiled module is the documented, inherent behaviour of compiling Erlang; the dependency being compiled is trusted to run code under Mix's model.

- **Output path `Path.join(compile_path, "#{module}.beam")` (lines 213, 241; and the delegated `Path.join(dest_dir, module <> "." <> …)` at erlang.ex:250) — traversal.**
  Confidence: high. `module` is derived from `Path.basename |> Path.rootname`, which cannot contain `/` (re-verified: `"a/b/../c.erl"` → `"c"`), and dep-only atoms never become digraph vertices (`:digraph.add_edge` returns `{:error, {:bad_vertex, _}}` for a missing vertex, which is ignored), so the interpolated name is always a real source module confined under `compile_path`/`dest_dir`. The delegated `extract_entries`/`module_from_artifact` (erlang.ex:245‑262) apply the identical `basename |> rootname` derivation, so the confinement holds in the delegated path too.
  Axes examined: slash-stripping of the module component in both files; vertex-existence gate on dep atoms. | Not examined: none material.

- **`erlc_options` / `erlc_include_path` / `erlc_paths` handling (lines 58‑84; delegated `assert_valid_erlc_paths` erlang.ex:222).**
  Confidence: high. All originate from `mix.exs` project config, which is operator-controlled and executes as code by design — not an attacker boundary. The `not is_list(erlc_options)` guard and `assert_valid_erlc_paths` raise cleanly on non-lists.

- **Delegated `read_manifest`/`write_manifest` (erlang.ex:265‑280) — `binary_to_term` on the manifest.** *(Upgraded from hedged "out of scope" caveat in first pass — now read and assessed.)*
  Confidence: high. `read_manifest` calls `:erlang.binary_to_term/1` **without `:safe`** on the manifest file, then validates the `{@manifest_vsn, data}` shape and `rescue`s any failure to `[]`. `binary_to_term` on untrusted bytes is a classic atom-exhaustion / resource sink, but the manifest lives under `Mix.Project.manifest_path()` in `_build/`, is written only by `write_manifest` (Mix itself), and is not reachable by the remote/archive attacker in this folder's threat model. No attacker boundary is crossed via the data this file hands to the delegated module (a list of `{status, file, beam_path}` derived from confined filenames).
  Axes examined: manifest provenance (Mix-written, `_build`-local), missing `:safe` flag, rescue coverage. | Not examined: none material to this file's inputs.

## Risks / caveats
- I could not read `String.to_unsafe_atom/1`'s source (this checkout contains only `elixir/lib/mix`; `String` is compiled into the installed runtime), so its behaviour is characterised by running it against the built OTP‑29 / Elixir toolchain rather than by source inspection; my UTF‑8/length conclusions are `verified` behaviourally, not read from source.
- The invalid-UTF‑8 filename reachability depends on the VM's filename encoding mode. I verified `+fnu` filters the file. An exotic locale/mode where `Path.wildcard` surfaces raw undecodable bytes *and* `binary_to_atom(_, :utf8)` rejects them could in principle re-open the `scan_sources` crash path, but Elixir emits a startup warning that such native encodings "may cause Elixir to malfunction," so this is a misconfiguration precondition rather than a live vector. This axis stays on `Not examined` for the first ruled-out entry.
- Behaviour of `:code.purge`/`:code.delete` (system-module hang vs survivable) was not re-run this pass; the trust-model objection makes the exact failure mode immaterial to the non-finding verdict.

## Did I verify this?
- **Delegated `Mix.Compilers.Erlang` module — read in full this pass** (erlang.ex:1‑376). This resolves the first-pass "single-file review" hedge. Verified that `compile_entries/6`, `do_compile/4`, `extract_entries/5`, `module_from_artifact/1`, `to_diagnostics/2`, `read_manifest/1` receive only filename-derived, `compile_path`-confined data from this file and introduce no new attacker-reachable sink. **verified (read).**
- **`to_unsafe_atom` length + UTF‑8 axes** — re-ran on sandbox: 256→`SystemLimitError`, 255→ok, `<<0xFF>>`→`ArgumentError`. **verified (executed).**
- **`Path.wildcard` filters invalid-UTF‑8 filenames under `+fnu`** — re-ran with a real `src/mod\xFF.erl` on disk; only `src/good.erl` returned. **verified (executed).**
- **`Path.basename |> Path.rootname` cannot yield a slash** — `"a/b/../c.erl"` → `"c"`. **verified (executed).**
- **Delegated parallel reducer lacks an `{:exit,_}` clause but the callback's only raise path (bare `:error` from `:compile.file`) is operator-config, not attacker-reachable** — verified by reading `do_compile` return-shape clauses (erlang.ex:282‑308). **verified (read).**
- `to_unsafe_atom`/`String` source itself — **not verified from source** (not in checkout); behaviour characterised by execution only.

## Dropped in re-review
None. Every prior ruled-out entry is preserved; the prior "out of scope" caveat on `Mix.Compilers.Erlang` has been upgraded to a verified ruled-out entry (`read_manifest` `binary_to_term`) plus cross-file confirmation folded into the existing entries.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Output path `Path.join(compile_path, "#{module}.beam")` (lines 213, 241) — traversal.** `module` is derived from `Path.basename |> Path.rootname`, which cannot contain `/`, and dep-only atoms never become digraph vertices (`:digraph.add_edge` returns `{:error, {:bad_vertex, _}}` for a missing vertex, which is ignored), so the interpolated name is always a real source module confined under `compile_path`.
