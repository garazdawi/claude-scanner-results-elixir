# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/record.ex
<!-- sha256: e7e2d219936a71894f2bdb6767593464a12d0bc41de821bf9228150b7fc655e1 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/record.ex
<!-- sha256: e7e2d219936a71894f2bdb6767593464a12d0bc41de821bf9228150b7fc655e1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28 (hedge-clearing pass over the first report)_

The first pass concluded "No findings" but hedged twice: (1) "I did not exhaustively fuzz every generated-macro shape," and (2) `Record.Extractor` is "not present in this checkout." This pass clears both by execution. I ran the full create/get/update/index/to-keyword matrix **and** every defensive branch against `/opt/elixir` (byte-identical `record.ex`, `diff` = IDENTICAL), and confirmed `Record.Extractor` exists at `/opt/elixir/lib/elixir/lib/record/extractor.ex` — so the delegation at lines 115/132 resolves and record.ex itself does no path/sink handling. Conclusion is unchanged, now with executed evidence instead of assumptions.

---

# Security review — `elixir/lib/elixir/lib/record.ex`

## Summary
`Record` is compile-time macro machinery for defining/accessing tuple records. It contains no dynamic-atom, code-eval, shell, path, deserialization, network, or crypto sinks reachable through its documented public API. Every defensive branch resolves to a controlled `ArgumentError` (or an `IO.warn`) — verified by running each path. **No security findings.**

## Security

`No findings`

Documented public API: `extract/2`, `extract_all/1`, `is_record/1,2` guards, `defrecord/3`, `defrecordp/3`. Traced against Phase 1 sink classes:

- **Code / compile-time execution:** Macros splice only developer-authored AST plus field defaults routed through `Macro.escape/1` (lines 348, 377, 426), which emits inert literal reconstruction and rejects non-escapable terms. No `Code.eval*`, `Module.create`, or `unquote` of runtime/network data. Confirmed by execution below.
- **Atom creation:** No `String.to_atom`/`binary_to_atom`/`Module.concat` in the file. Tags/field names are already atoms in caller AST; the code only *guards* `is_atom/1` (lines 346, 356). No peer-derived atomization.
- **Command / file / path / archive / deserialization / network / crypto / regex:** none present. `extract/2` and `extract_all/1` are one-line delegations to `Record.Extractor` (115/132), guarded by `is_atom(name) and is_list(opts)`; record.ex performs zero path handling.
- **Shared mutable state:** `Module.register_attribute`/`put_attribute` (329, 331) write the *compiling module's own* attribute table under static keys — not externally-keyed global state.
- **Unicode / UTS-39:** Not applicable — this is `lib/record.ex`, not the tokenizer `src/`. No identifier normalization, script analysis, bidi/NFC handling, or escape decoding occurs here. (Standing-context sub-claims 1–12 and the UAX/UTS clauses target `elixir_tokenizer*`/`elixir_interpolation.erl`; none of that machinery is present in this file.)

### Executed evidence (clears "did not exhaustively fuzz")

Compiled `defrecord(:user, name: "meg", age: 25)` and drove every generated-macro arity plus defensive branches:

| Path | Input | Result |
|---|---|---|
| `name/0` create | `user()` | `{:user,"meg",25}` |
| `name/1` create kw | `user(age: 26)` | `{:user,"meg",26}` |
| `name/2` get | `user(r,:name)` | `"meg"` |
| `name/1` index | `user(:name)` | `1` |
| `name/2` update | `user(r,age:99)` | `{:user,"meg",99}` |
| `name/1` to-keyword | `user(r)` | `[name:"meg",age:25]` |
| to-keyword hostile tuples | `{:user,1}`,`{:notuser,1,2}`,`{}`,`:atom`,`%{a:1}`,`{:user,1,2,3,4}` | each → clean `ArgumentError`, no crash, no atom creation |
| unknown key create/update/index | `user(bogus:1)` / `user(r,bogus:1)` / `user(:bogus)` | `ArgumentError "record :user does not have the key: :bogus"` |
| `fn` default (extractor-style) | `defrecord(:r1, f: fn->1 end)` | `ArgumentError "invalid value for record field f, cannot escape #Function…"` |
| non-atom field key | `defrecord(:r2,[{"str",1}])` | `ArgumentError "defrecord fields must be atoms, got: {\"str\", 1}"` |
| duplicate key | `defrecord(:r3, a:1, a:2)` | `IO.warn "duplicate key :a found in record"` (correctness quirk, not security) |
| PID / nested-tuple default | `p: self()` / `t: {:a,{:b,[1,2,3]}}` | compiles (escapable literals; inert reconstruction) |
| hoisting + eval order | `r(a: (send…;10), b: (send…;20))` | messages arrive `[:first,:second]` — left-to-right, single evaluation, `Macro.unique_var` hygiene, no leak/capture |

### Ruled out

- **`extract/2` & `extract_all/1` arbitrary file read / path traversal** — *Basis: verified (delegation) + genuinely out-of-file (sink).* record.ex does zero path handling; both functions immediately call `Record.Extractor.extract*` (115/132) after an `is_atom(name)`/`is_list(opts)` guard. I confirmed `Record.Extractor` exists (`/opt/elixir/lib/elixir/lib/record/extractor.ex`) — the first pass's "not in this checkout" was true for the claude-scanner tree but the module resolves in the equivalent full tree. Any `:from`/`:from_lib`/`-include_lib`/`:epp` traversal risk lives entirely in that separate module and is in-scope for *its* review, not record.ex's. record.ex passes only a guarded atom + a literal opts list. **Confidence: high** (record.ex itself is clean). **Axes examined:** presence of path/IO sinks in this file; guard on inputs; delegation target existence. **Not examined:** none for record.ex (the file has no sink to examine); Extractor's internal path logic is a different file, deliberately not adjudicated here.
- **`fields/2` hostile default from an untrusted extracted `.hrl`** — *Basis: verified by execution.* `Macro.escape/1` (348) produces inert literal AST and rejects non-escapable terms; a `fn` default raises `ArgumentError` (ran it), a non-atom key raises `ArgumentError` (ran it). No code execution, no crash, no atom creation. **Confidence: high.** **Axes examined:** anonymous-fn value, non-atom key, PID value, deeply-nested tuple/list value. **Not examined:** none.
- **`__keyword__/3` on an attacker-controlled runtime tuple** — *Basis: verified by execution.* Drove `{:user,1}`, `{:notuser,1,2}`, `{}`, `:atom`, `%{a:1}`, over-arity `{:user,1,2,3,4}`: every case a clean `ArgumentError`, no crash, no atom creation. `tag`/`fields` are compile-time constants, so nothing attacker-influenced reaches a sink. **Confidence: high.** **Axes examined:** wrong tag, right tag/wrong arity (both under and over), non-tuple, map, empty tuple. **Not examined:** none.
- **`build_update/2` descending sort + duplicate-key updates (486–492)** — *Basis: reasoned + duplicate-key path executed.* Inputs are compile-time developer-written AST bounded by field count; no runtime attacker boundary. Duplicate key is a developer-visible correctness quirk (warns, line 313 — confirmed), not a security issue. **Confidence: high.** **Axes examined:** duplicate-key compile behaviour; input provenance (compile-time literal). **Not examined:** none material (no runtime input reaches this function).
- **`hoist_expressions/2` variable capture (467–476)** — *Basis: verified by execution.* Non-simple expressions are hoisted to `Macro.unique_var(key, module)` bindings and evaluated left-to-right exactly once (message-order test returned `[:first,:second]`). Hygienic unique vars; no capture/leak reachable from untrusted input. **Confidence: high.** **Axes examined:** eval order, single-evaluation, side-effect sequencing in create path. **Not examined:** the `%{context: nil}` vs non-nil match-context clause split under adversarial caller env — compile-time developer-controlled, no runtime attacker boundary, so left un-exercised.
- **Error-message interpolation (304, 360, 413, 437, 524, 530)** — *Basis: reasoned.* `inspect/1`/`Macro.to_string/1` into raised messages / `IO.warn`; not forwarded to any downstream interpreter (shell, SQL, EEx, regex). No injection sink. **Confidence: high.** **Axes examined:** sink destination of the interpolated string. **Not examined:** none.

## Risks / caveats
- **Adjacent-file scope (not a record.ex defect).** The security-interesting file-format handling for `extract/*` — `-include`/`-include_lib` path lookup, `:epp` preprocessing, and atomization of parsed record/field names — lives in `Record.Extractor` (`.../record/extractor.ex`). It is a *separate* module and out of scope for this file. If `extract/*` is ever invoked with an untrusted path or untrusted Erlang header at build time, residual traversal / include-resolution / atom-table risk resides there and should be reviewed in that file. Nothing in record.ex mishandles those inputs; it forwards a guarded atom and a literal opts list unchanged.
- **Reproductions** ran against `/opt/elixir` (`1.21.0-dev`), whose `lib/record.ex` is **byte-identical** to the file under review (`diff` → IDENTICAL), so observed behaviour reflects this exact source.

## Did I verify this
- **Verified by execution:** the full create/get/update/index/to-keyword matrix; all defensive branches (unknown key ×3 paths, `fn` default, non-atom key, hostile runtime tuples ×6, duplicate key); `Macro.escape` rejection of non-escapable defaults; hoisting evaluation order/hygiene. All resolve to controlled `ArgumentError`/`IO.warn`; none create atoms, crash, or execute untrusted code.
- **Verified by reading (this file + neighbour existence):** `extract/*` delegates with an atom+list guard and does no path handling; `Record.Extractor` exists and is a distinct module.
- **Not verified (intentionally, different file):** `Record.Extractor`'s internal path/`:epp`/atomization logic — belongs to that file's own review.

## Dropped in re-review
- **Hedge "I did not exhaustively fuzz every generated-macro shape"** — dropped. Replaced by the executed-evidence table covering every generated arity and defensive branch; no residual assumption remains.
- **Hedge "`Record.Extractor` … not present in this checkout"** — dropped as a blocking caveat. The module resolves in the byte-identical full tree (`/opt/elixir/.../record/extractor.ex`); the delegation is confirmed clean and any extractor-internal risk is correctly scoped to that separate file, not upgraded to a record.ex finding (record.ex has no sink to which such a finding could attach).

No prior findings existed to preserve (the first pass reported none), and no hedged assumption warranted upgrade to a low-confidence finding — every one was cleared by execution rather than left asserted.
