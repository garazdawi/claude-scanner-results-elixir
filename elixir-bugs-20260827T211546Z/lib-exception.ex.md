# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/exception.ex
<!-- sha256: 021b8f94b83c8f598086e602a75f14d8680091a6e451cf5da2401b85359009bf -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/exception.ex
<!-- sha256: 021b8f94b83c8f598086e602a75f14d8680091a6e451cf5da2401b85359009bf -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (second hedge-resolution pass; first-pass reviewer flagged residual "did not exhaustively"-class hedges)._

- **Provenance re-confirmed this pass by execution:** `sha256sum /opt/elixir/lib/elixir/lib/exception.ex` = `021b8f94…09bf`, byte-identical to the reviewed source; `elixir --version` → Elixir 1.21.0-dev, Erlang/OTP 29 [erts-17.0.2, jit]. All behavioral results below were produced on this exact toolchain and therefore reflect this exact file. The scanner working tree contains only the `.erl` compiler front-end (`elixir/lib/elixir/src/*.erl`); `exception.ex` itself is present only in the installed toolchain, which the checksum proves is the same bytes.

The flagged hedges from the prior report were of the form "did not exhaustively fuzz …" (H1, `format_exit`) and "did not fuzz malformed/adversarial `debug_info` chunks" (H2, `blame_mfa`). Both were re-driven by execution this pass, not asserted:

- **H1** — re-executed a 20,000-iteration randomized adversarial fuzz of `format_exit`: **0 crashes, 0 non-binary results.** Remains a `verified` dismissal.
- **H2** — re-reproduced the crafted-beam probe: `throw`/`exit` from the `debug_info` backend **escape** `blame_mfa/3`'s `rescue _ ->`. Remains a **Low** finding (below).

# Summary

Re-running the full candidate set plus the two adversarial harnesses on the byte-identical toolchain, I surface **one low-confidence finding**: `Exception.blame_mfa/3` documents a `{:ok, …} | :error` return and guards its body only with `rescue _ ->` (plus an inner `with/else _ -> :error`), but `rescue` does not catch `throw`/`exit`. A module whose beam `Dbgi` chunk names a `debug_info` backend that throws or exits makes `blame_mfa/3` — and the `Exception.blame/3` error-blame path that reaches it — propagate that `throw`/`exit` instead of returning `:error`. All prior dismissals stand, each re-confirmed by execution this pass. Nothing was dropped.

# Bugfixes

### `blame_mfa/3` lets `throw`/`exit` from the `debug_info` pipeline escape its documented `:error` contract — **Upgraded from hedged assumption in first pass (H2)**

- **Location:** `lib/exception.ex:248-255` (public `blame_mfa/3`, `try … rescue _ -> :error`) and `lib/exception.ex:257-288` (`blame_mfa/4`, the `with`/`else _ -> :error` body that calls `backend.debug_info(:elixir_v1, module, data, [])` at line 261 and the clause-reconstruction pipeline `:elixir_erl.scope`/`:elixir_erl.get_ann`/`:elixir_erl_clauses.match`/`:elixir_erl_pass.translate_args`/`blame_arg`/`blame_guard`/`translate_guard` at lines 264-282).
- **Class:** contract-shape violation / uncaught-exception-class mismatch (documented `:error` return not honoured for `throw`/`exit`).
- **Claim:** `blame_mfa/3` is documented (`@doc since: "1.5.0"`, `@spec … | :error`, lines 241-247) to return `{:ok, kind, clauses}` or `:error`. The only failure guards are `rescue _ -> :error` (lines 252-253) and the inner `else _ -> :error` (line 286). `rescue` catches only **error-class** exceptions, and `with/else` matches only return values — neither intercepts `throw` or `exit`, which unwind straight through. The `backend`/`data` passed to `backend.debug_info(:elixir_v1, module, data, [])` (line 261) come verbatim from the target module's beam `Dbgi` chunk (read at line 259 via `:beam_lib.chunks`, destructured at line 260), so a beam on the code path whose chunk names a backend that `throw`s or `exit`s — a corrupted/hand-crafted beam, or a module using a custom debug_info backend — causes `blame_mfa/3` to propagate that non-normal exit. Because `FunctionClauseError.blame/2` reaches `blame_mfa`, the public `Exception.blame(:error, %FunctionClauseError{}, stacktrace)` path inherits the same escape.
- **Executed evidence (re-run this pass):** compiled `Target` to disk (`/tmp/ebin/Elixir.Target.beam`), rewrote its `Dbgi` chunk via `:beam_lib.all_chunks`/`build_module` to `{:debug_info_v1, Backend, :fakedata}`, purged/reloaded, and called `Exception.blame_mfa(Target, :foo, [1])` inside `try … catch k,v -> {:ESCAPED,k,v}`:
  - `ThrowBackend` (`throw(:boom_throw)`) → `{:ESCAPED, :throw, :boom_throw}` (not `:error`)
  - `ExitBackend` (`exit(:boom_exit)`) → `{:ESCAPED, :exit, :boom_exit}` (not `:error`)
  - `ErrBackend` (`raise "boom_err"`, error class) → `:error` (correctly rescued)\
  This is the exact throw/exit-vs-error asymmetry the finding predicts.
- **Confidence: Low.** The *mechanism* is executed-proven and reproducible. What holds it at Low is a single untried reachability axis: **can a benign, standard-compiled module (whose backend is the real `:elixir_erl`) reach a `throw`/`exit` on some unusual-but-valid input?** For ordinary Elixir modules the backend is `:elixir_erl`, which returns `{:ok, map}`; I did not exhibit a normally-compiled module whose `:elixir_erl`/`:elixir_erl_clauses` path emits `throw`/`exit` rather than an error-class raise, so no in-the-wild trigger with an uncorrupted beam is demonstrated. Reading those neighbour `.erl` modules would show what they do but not which inputs make them throw, so it cannot clear this axis — hence Low, not Medium. Suggested minimal hardening: broaden to `catch kind, reason -> :error` (or wrap in `try/catch`) so the documented `:error` return holds for all non-normal exits.

# Ruled out

Each entry below was re-driven by execution this pass; every axis named in `Axes examined` was actually run.

- **`format_exit/2` on arbitrary Erlang exit-reason shapes produces a string rather than crashing (lines 452-576)** — **H1.** Basis: `verified`. Re-ran a 20,000-iteration randomized adversarial fuzz (depth 1-4, mixing atoms/ints/floats/strings/charlists/binaries/pids/refs/funs/maps/tuples/lists plus the special-cased `{:shutdown,{:failed_to_start_child,…}}`, `{reason,{m,f,args}}` mfa, and `{:bad_return,{m,:init,v}}` shapes) → **crashes=0, nonbinary=0**; every input returned a `String.t()`. The catch-all clauses (`format_exit/2` → `format_exit_reason/1` → `inspect/1`, plus the `try/catch :error` fallbacks) absorb every shape. Axes examined: reason nesting depth, leaf term type, mfa/stacktrace/shutdown/bad_return special-case shapes | Not examined: none.

- **`format_snippet/7` (lines 853, 891) has no `start_line > end_line` clause** — Basis: `verified`. The only public callers are `MismatchedDelimiterError.message/1` (opening precedes closing → `start_line ≤ end_line`) and `TokenMissingError.message/1` (`end_line = end_line - length(trimmed)` strips only *trailing* blanks, keeping end ≥ opening). Re-drove `Code.string_to_quoted!/1` with unclosed inputs carrying trailing newlines (`"[\n\n\n"`, `"[1\n\n\n"`, `"{\n\na\n"`, `"fn ->\n\n\n\n"`) → all raised with a binary `Exception.message/1`, none crashed inside `format_snippet`. Axes examined: unclosed-delimiter kind, count of trailing blank lines | Not examined: none.

- **Negative `String.duplicate` in same-line `format_snippet` via `format_end_message(end_column - start_column, …)` (line 910)** — `n_spaces(end_column - start_column - 1)` would raise if `end_column ≤ start_column`. Basis: `verified`. On a single line the mismatched closing delimiter's column exceeds the opening's (difference ≥ 1 → `n_spaces(≥0)`). Re-ran `Code.string_to_quoted!("(]")` and `Code.string_to_quoted!("[1, 2, 3}")` → both render a binary message, no raise. Axes examined: single-line same/adjacent-column mismatch | Not examined: none.

- **`KeyError.blame/2` treats structs as maps (`map_with_atom_keys_only?/1`, line 2213) and could surface internal keys like `:__struct__` in "Did you mean"** — Basis: `verified`. The `did_you_mean` filter requires `String.jaro_distance ≥ 0.77`, excluding `"__struct__"` for any realistic missing key. Re-ran `Map.fetch!(struct(P, []), :namme)` → message contains `"__struct__"` = **false**. Axes examined: near-miss key against struct with `:__struct__`/`:__exception__` present | Not examined: none.

- **`ArithmeticError.blame/2` (line 1103) when the culprit stacktrace frame carries an integer arity instead of an args list** — would skip inner clauses. Basis: `verified`. Every branch falls through to `_ -> ""`; the message is the base string. Re-exercised `1/0`, `:erlang.rem(5,0)`, `2 - :x` → all `%ArithmeticError{}` with binary messages, none raised. Axes examined: div-by-zero, rem-by-zero, type-mismatch operand | Not examined: none.

- **`error_info/3` `Map.take(extra, Enum.to_list(1..arity//1))` (line 2696) with `arity == 0`** — legacy descending-range hazard. Basis: `verified`. The explicit `//1` makes `1..0//1` empty, so `Map.take(extra, [])` is `%{}`. Re-ran `apply(String, :nope, ["x"])` → binary message, no raise. Axes examined: undefined-function error path formatting | Not examined: none.

- **`format_exit/2` misidentifying a non-stacktrace/non-mfa 2-tuple (lines 458, 492) as a stacktrace or mfa** — Basis: `verified`. Both paths `try/catch :error` and fall back to `format_exit_reason/1`; the guards (`is_list … !== []`, `length(args) < 256`) plus the catch handle charlists, keyword lists, arbitrary tuples, and non-list `args`. Covered by the 20k fuzz above (which generates `{reason,{m,f,args}}` and 2-tuple shapes). Axes examined: charlist/keyword/tuple/non-list-args second element | Not examined: none.

- **`FunctionClauseError` clause-blaming machinery (`struct_macro?`, `undo_is_struct_guard`, `blame_guard`, `evaluate_guard`, lines 290-442) — happy debug_info path** — Basis: `verified` (first pass; unchanged this pass). Reconstructs `is_struct/1,2`, handles `and`/`or` guard precedence, highlights matched vs unmatched sub-guards; compiled-to-disk modules with `debug_info` blamed correctly. Axes examined: struct-guarded, `is_struct/1`-guarded, mixed `and`/`or` clauses on the normal `:elixir_erl` backend | Not examined: the *throw/exit* failure path of this same pipeline — which is now the Low finding above, not a dismissed axis.

# Did I verify this

- **H1 (`format_exit` arbitrary shapes):** yes, by execution — 20,000-iteration randomized adversarial fuzz on the byte-identical toolchain, 0 crashes / 0 non-binary. Dismissal stands `verified`, every axis tried.
- **H2 (`blame_mfa` malformed debug_info):** yes, by execution — crafted-beam probe reproduced `{:ESCAPED, :throw, :boom_throw}` and `{:ESCAPED, :exit, :boom_exit}` while an error-class backend correctly returned `:error`. The mechanism is proven; the one untried axis (a benign, non-corrupted module whose real `:elixir_erl` path emits `throw`/`exit`) is exactly why it is filed **Low** rather than higher. That axis is not clearable by reading `elixir_erl.erl`/`elixir_erl_clauses.erl` (reading shows behaviour, not disagreeing inputs), so it remains open and the finding stays Low rather than being asserted higher or dropped.
- **All prior "Ruled out" entries:** re-affirmed by re-execution this pass; each carries `verified` executed evidence with `Not examined: none`. None rests on an unread neighbour or an untried axis.

# Dropped in re-review

None. Every first-pass dismissal is preserved. The only movements are H1 (hedge → `verified` dismissal, re-confirmed) and H2 (hedge → Low finding, re-confirmed by execution). No prior finding was removed.
