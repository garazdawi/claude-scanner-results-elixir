# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code.ex
<!-- sha256: c487ddbf6aab8916d6a33f11cd0e64a33ab3d0676381f3d2ab0dc1b028196ad2 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/code.ex
<!-- sha256: c487ddbf6aab8916d6a33f11cd0e64a33ab3d0676381f3d2ab0dc1b028196ad2 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass over the 2026-08-27 first pass)_

## Summary
`code.ex` is the public `Code` module (compile/eval/parse/format/doc entry points). Its documented API and all doctests behave as specified. This pass re-ran the first-pass evidence on the local toolchain (Erlang/OTP 29, Elixir 1.21.0-dev) and additionally exercised the axes the first pass had only asserted. Two items change status: the `compiler_options/1` error-asymmetry finding is re-confirmed by execution, and the previously-declined `binary_to_term`-without-`:safe` caveat is **upgraded to a low-confidence finding** (deserialization sink on attacker-supplied file input). No ruled-out entry was moved to `verified` on the strength of a neighbour read alone.

## Bugfixes

#### `CSCAN-code-compiler-options-opaque-error`
- **Finding ID** — `CSCAN-code-compiler-options-opaque-error`
- **Severity** — Low
- **Confidence** — medium
- **Location** — `code.ex:1705-1711` (`compiler_options/1`) via `code.ex:1726` (`get_compiler_option/1`)
- **Issue** — `compiler_options/1` computes the previous value with `get_compiler_option(key)` (line 1707) *before* calling `put_compiler_option(key, value)` (line 1708). `get_compiler_option/1` (lines 1726-1734) has only the `key in @available_compiler_options` clause plus a `:warnings_as_errors` deprecation clause — no catch-all. For an unknown key it raises a bare `FunctionClauseError` naming `Code.get_compiler_option/1`, a function the caller never invoked. The singular setter `put_compiler_option/2` (catch-all at lines 1947-1949) raises a clear `"unknown compiler option: :bad_key"` for the same bad key; the plural form loses that message.
- **Class** — Missing clause / non-total function (with error-message asymmetry)
- **Impact** — A caller passing a mistyped/unsupported option through the documented public API `Code.compiler_options([bad_key: 1])` gets a cryptic error pointing at an internal helper rather than the actionable "unknown compiler option" message the analogous singular call produces. Cosmetic/diagnostic only — no wrong result, no state corruption.
- **Evidence** — `verified` (re-executed this pass on Erlang/OTP 29, Elixir 1.21.0-dev):
  - `Code.compiler_options([bad_key: 1])` → `** (FunctionClauseError) no function clause matching in Code.get_compiler_option/1`
  - `Code.put_compiler_option(:bad_key, 1)` → `** (RuntimeError) unknown compiler option: :bad_key`
  - Nuance found while re-verifying: `Code.compiler_options([warnings_as_errors: true])` does *not* raise — it returns `%{warnings_as_errors: :ok}`, because both `get_compiler_option(:warnings_as_errors)` (line 1731) and `put_compiler_option(:warnings_as_errors, _)` (line 1919) are deprecation stubs returning `:ok`. So the asymmetry is specific to keys with no clause at all; the finding stands for the general bad-key case.
- **Suggested fix** — Give `compiler_options/1` a validation/error path consistent with `put_compiler_option/2`, e.g. guard the loop with `key in @available_compiler_options` (or wrap the `get_compiler_option` call) and raise the same `"unknown compiler option: #{inspect(key)}"` message for unrecognized keys before touching config.

#### `CSCAN-code-fetch-docs-unsafe-binary-to-term`  *(Upgraded from hedged assumption in first pass)*
- **Finding ID** — `CSCAN-code-fetch-docs-unsafe-binary-to-term`
- **Severity** — Low
- **Confidence** — low
- **Location** — `code.ex:2382-2387` (`load_docs_chunk/1`), reachable from `fetch_docs/1` binary-path clause (`code.ex:2340-2341` → `fetch_docs_from_beam/1` 2356-2370 → line 2359) and from `fetch_docs_from_chunk/1` (2372-2379, `File.read` on a path derived from the beam location).
- **Issue** — `load_docs_chunk/1` calls `:erlang.binary_to_term(bin)` **without the `:safe` option** on bytes taken from a user-supplied `.beam`/`.chunk` file. `Code.fetch_docs("/attacker/controlled/path")` reaches this with fully attacker-controlled file content: `fetch_docs("...")` → `String.to_charlist` → `:beam_lib.chunks/2` extracts the `Docs` chunk → `load_docs_chunk` deserializes it. The `rescue` at 2384-2386 only catches decode exceptions; it does not constrain what terms are materialized before/without an exception.
- **Class** — Unsafe deserialization (`binary_to_term` without `:safe`)
- **Impact** — Deserializing an untrusted term can materialize arbitrary atoms (atom-table growth toward the node-wide limit → DoS) and arbitrary large/nested terms. It does not by itself execute code, but the EEP-48 doc term is later traversed by doc tooling, widening the blast radius. This is longstanding documented EEP-48 behavior and not a regression, which is why the first pass declined it; re-review upgrades it to a low-confidence finding rather than leaving it as an unverified dismissal, because the sink (attacker file → non-`:safe` `binary_to_term`) is real and reachable through the public API.
- **Evidence** — `verified` that the code path exists and is non-`:safe` (read at lines 2340-2387). Not executed against a crafted malicious chunk (would require building an adversarial `.beam`), hence low confidence on exploitability/severity.
- **Suggested fix** — If the doc term shape permits, pass `[:safe]` to `binary_to_term`; otherwise document the trust requirement on `fetch_docs/1` for arbitrary paths. Note `:safe` may reject legitimate chunks containing atoms not present on the node, so this needs validation against real doc chunks before applying.

## Risks / caveats
- The file is overwhelmingly documentation plus thin delegations to `:elixir`, `:elixir_config`, `:elixir_code_server`, `:elixir_compiler`, `:code`/`:beam_lib`, and `Module.ParallelChecker`. Deep bugs inside those callees remain out of scope, but for this pass I confirmed by execution that `code.ex`'s own argument marshalling and error surfacing behave as claimed on the reachable paths that mattered to the hedges (compiler-option dispatch, eval third-arg dispatch, `:code.del_path` return handling). Where I could not *execute* an axis, I left the corresponding dismissal at `asserted` (see "Did I verify this").
- All executable doctests were re-run in the first pass and reproduced exactly: `string_to_quoted/2`, `eval_string/3`, `string_to_quoted_with_comments/2`, `ensure_all_loaded/1`, `fetch_docs/1`. None mismatch the code; not re-run this pass (no code change).

### Ruled out

- **`eval_string`/`eval_quoted` default-arg + env-vs-list dispatch** (lines 633-639, 1186-1192) — both clauses (`%Macro.Env{}` and `is_list(opts)`) correctly thread `validate_binding`/`env_for_eval`. A non-list, non-env third argument raises `FunctionClauseError` naming `Code.eval_string/3` — the function the caller actually invoked (so, unlike the `compiler_options` case, the error is *not* misdirected to an internal helper). Spec violation (misuse), not a reachable bug.
  Basis: `verified`. Axes examined: third-arg type — tried `Code.eval_string("1", [], :bad)` (atom) → `FunctionClauseError`, and `Code.eval_string("1", [], %{a: 1})` (non-`Env` map/struct) → `FunctionClauseError`; error names the public `/3` head, not a private helper. | Not examined: none.

- **`with_diagnostics(opts \\ [], fun)` default on first of two params** — generates `/1` and `/2`; `with_diagnostics(fn -> … end)` binds `opts=[]`, `fun=fn`, and the `after` block restores/erases `:elixir_code_diagnostics` from the captured prior value.
  Basis: `verified`. Axes examined: arity dispatch + process-dictionary restore — called `Code.with_diagnostics(fn -> IO.warn("x") end)`, inspected returned diagnostics and confirmed the process key was restored. | Not examined: none.

- **`next_eol_count/2` clause ordering (`?\n` before `?\r,?\n`)** — a `\r\n` pair still matches the two-element clause because the single-`?\n` head cannot match a leading `?\r`; lone `?\r` falls through to the catch-all, consistent with documented `\r\n`→`\n`.
  Basis: `verified`. Axes examined: newline-shape counting — the `":foo # :bar"` and multi-line comment doctests exercise the counting and reproduce exactly. | Not examined: none.

- **`min(previous_eol_count, last_comment_distance)` with `:infinity`** — `last_comment_distance/2` returns the atom `:infinity` for an empty comment list; Erlang term ordering ranks atoms above numbers, so `min(int, :infinity)` yields the integer ("no cap").
  Basis: `verified`. Axes examined: term-ordering of atom-vs-integer in `min/2` — comment doctests produce the documented `previous_eol_count` values 2 and 3. | Not examined: none.

- **`delete_path/1` return handling** (lines 540-548) — `:code.del_path/1`'s documented return set (`true`/`false`/`{:error,:bad_name}`) is covered; `delete_path` always feeds it a charlist via `to_charlist(Path.expand(path))`, so the `function_clause`-on-non-list branch of `del_path` is unreachable from here.
  Basis: `asserted`. Axes examined: return shape for an absent path — `code:del_path("/no/such/dir")` → `false` (boolean branch taken); non-charlist input — `code:del_path(123)` → `function_clause`, confirming that branch exists but is unreachable because the caller always passes a charlist. | Not examined: exhaustive enumeration of which charlist inputs yield `{:error,:bad_name}` versus any undocumented shape — this would require reading OTP `code.erl`, which by rule does not clear the axis, and I could not construct an input that produced a shape outside the handled set, so I cannot *prove by execution* that no other shape exists. Stays `asserted`.

- **`get_beam_and_path/1` binding of `beam` vs `filename`** (lines 2344-2352) — `{beam, filename}` is returned and destructured as `{bin, beam_path}` in `fetch_docs/1`; `bin` (object code) is passed to `fetch_docs_from_beam`, `beam_path` used only for `Path.expand`, matching `:code.get_object_code/1`'s `{Module, Binary, Filename}` tuple order.
  Basis: `verified`. Axes examined: tuple-position binding — `Code.fetch_docs(Atom)` returns the expected `:docs_v1` tuple and correct module-doc line (first pass). | Not examined: none.

### Did I verify this
- **Verified by execution (this pass):** the `compiler_options` finding (both commands re-run), the `warnings_as_errors` non-raising nuance, the eval third-arg dispatch axis (atom + map), and the `del_path` boolean/absent-path axis.
- **Verified by reading (this pass):** the `load_docs_chunk` → `binary_to_term` sink reachability from `fetch_docs/1`'s public binary-path clause.
- **Asserted (could not clear by execution):** the claim that `:code.del_path/1` produces no return shape beyond `true`/`false`/`{:error,:bad_name}` for charlist input. I tried the reachable and one unreachable input; enumerating the full OTP contract would require reading `code.erl`, which does not clear the axis. Left `asserted` deliberately.

### Dropped in re-review
- None. Every first-pass finding is preserved; the first-pass `binary_to_term` caveat was upgraded (not dropped) into `CSCAN-code-fetch-docs-unsafe-binary-to-term`.
