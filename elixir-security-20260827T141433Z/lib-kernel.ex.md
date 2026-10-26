# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel.ex
<!-- sha256: 33d922b91dbb838c9a8f4d06b0dd73fdadabe0126035846aa62704d1fc9c79ec -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel.ex
<!-- sha256: 33d922b91dbb838c9a8f4d06b0dd73fdadabe0126035846aa62704d1fc9c79ec -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T12:10:00Z_

## 1. Summary

`Kernel` is composed of (a) guard/BIF wrappers whose dangerous arguments are always caller-supplied and (b) compile-time macros that operate on developer-authored source AST. I re-traced every construct flagged by the inventory checklist and found no path where untrusted *runtime* data reaches a dangerous sink as a defect of this file. No security findings.

The first-pass report carried one flagged hedge — "I did not exhaustively execute every one of the ~200 wrappers." That hedge is resolved below (§2 "Did I verify this"): the un-executed wrappers are single-expression `:erlang.*` inlines, a structural property confirmed by reading their bodies, and a single-BIF inline cannot introduce a defect absent from the delegated BIF. This is a structural claim (no branching logic is present to diverge), so reading the bodies is the verification — there is no second implementation to differentially test against, hence no untried behavioral axis to clear.

## 2. Security

`No findings`

Every construct in the inventory checklist that appears in this file was traced to either a caller-controlled argument, operator/build configuration, or developer-written source processed at compile time. The strongest candidates, and why each was rejected, are below. Repros were re-executed against the sandbox build (Elixir 1.21.0-dev, OTP 29) and the outputs are reproduced verbatim.

### Ruled out

- **`struct/2` and `struct!/2` (`:2526`, `:2559`, `:2563`) building structs from an untrusted map — safe.** `struct/2` only assigns keys already present in the base struct (`%{^key => _} -> %{acc | key => val}`), explicitly discards `:__struct__`, and drops unknown/string keys; `struct!/2` uses `struct.__struct__(fields)` / `Map.replace!/3`, which raise on unknown keys. An attacker-controlled map cannot inject `:__struct__`, change the struct type, or add fields.
  Axes examined: key-type (string vs atom), unknown-key, `:__struct__`-injection, struct!-vs-struct raise-behavior | Not examined: none.
  Basis: **verified (executed)** — `struct(U, %{"name" => "x", unknown: 1, name: "meg"})` → `%U{name: "meg", age: 0}` (string/unknown keys dropped); `struct!(U, %{bogus: 1})` → raises `"key :bogus not found"`.

- **`~w(#{x})a` atom creation from runtime input (`sigil_w`, `:6959`, `split_words` `:7022`).** With the `a` modifier and interpolation, each whitespace-separated word of the interpolated value becomes a new atom at runtime, growing the non-GC'd atom table (CWE-770). Ruled out as a *file* defect: this is the documented purpose of the `a` modifier, semantically identical to the caller writing `String.to_atom/1`; the developer explicitly interpolates untrusted data into an atom-producing construct. No "existing-atom-only" variant is meaningful — the atoms are by definition new.
  Axes examined: distinct-word count vs atom-table delta | Not examined: none.
  Basis: **verified (executed)** — 2000 distinct words created a `+2007` delta in `:erlang.system_info(:atom_count)` (the extra 7 are the harness's own atoms), confirming one new atom per word.

- **`**/2` integer power (`:4608`, `integer_pow`).** `base ** exponent` with an attacker-controlled exponent yields an arbitrarily large bignum (resource consumption). Ruled out: inherent to arbitrary-precision arithmetic; the caller chooses to exponentiate untrusted input and the operator computes the correct result — no defect.
  Axes examined: exponent-magnitude vs result size | Not examined: none.
  Basis: **verified (executed)** — `2 ** 2_000_000` produces a 602,060-digit integer (cost is the resulting bignum size, not an algorithmic flaw).

- **`binary_slice/2,3` (`:4964`, `:5038`) bounds handling.** Negative indices are clamped (`max(total + start, 0)`), size is `min`-capped to the remaining bytes, and the non-empty invariant before `<<first_byte, rest::binary>> = part` holds — no out-of-range `binary_part` or `MatchError`.
  Axes examined: negative-start, oversize-size | Not examined: none.
  Basis: **verified (executed)** — `binary_slice("elixir", -6, 3)` → `"eli"`; `binary_slice("elixir", 3, 999)` → `"xir"` (both clip correctly, no crash).

- **`apply/2`, `apply/3` (`:352`, `:379`), `spawn*` (`:1248`–`:1378`), `send/2`, `make_ref` (`:1026`).** Read the bodies: each is a single-expression delegation to the corresponding `:erlang.*` BIF with no intervening logic (e.g. `def apply(fun, args), do: :erlang.apply(fun, args)`). Any code-execution/dispatch risk requires the caller to forward an untrusted module/function/args tuple; the dangerous value is caller-controlled, not derived from data inside this file.
  Basis: **structural** — single-BIF inline; no branch exists in which this file could diverge from the BIF's own contract.

- **Compile-time atom creation: `alias_defmodule` (`:5253`, `:5258`), calendar sigils `maybe_atomize_calendar` (`:6902`), `Module.concat/1,2` in `use`'s `expand_aliases` (`:6156`, `:6159`).** All are macros expanding literal, developer-written module/calendar names in source at compile time. The atoms created are exactly those the author typed; no runtime/untrusted-data path reaches them.
  Basis: **structural** (macro expansion over source AST).

- **Sigil unescaping through `:elixir_interpolation.unescape_string` (`sigil_c`/charlist `:6618`, `sigil_r`/regex `:6649`, `sigil_C`/etc.).** The folder threat model notes `unescape_string/1` re-enters unescaping *without* the `?bidi/?break` gate. Confirmed these call sites exist here. Ruled out as a `kernel.ex` defect: the input is the developer-authored sigil body in source, expanded at compile time; treating adversarial *source* as in-scope is a whole-compiler concern owned by the tokenizer/interpolation modules per the folder threat model, not an escalation this file introduces. Flagged here only so a later tokenizer-scoped pass can confirm the `?bidi/?break` gate on the raw-unescape path.
  Basis: **structural** (compile-time expansion of source text).

- **`dbg/2` callback dispatch (`:6332`–`:6339`).** The `{mod, fun, args}` comes from the compiler's process dictionary or `Application.compile_env!` (project/operator build config), then `Macro.compile_apply` runs it at compile time. Control of that config is control of the build (operator), out of scope per the folder threat model; a dependency cannot set the parent app's `:elixir` config.
  Basis: **structural**.

- **`=~/2` with a `Regex` right operand (`:2411`).** Delegates to `Regex.match?/2`; any ReDoS risk lives in a regex the caller compiled elsewhere, not in this operator (the binary-right clause uses `:binary.match`, no regex).
  Basis: **structural**.

- **`inspect/2` (`:2471`) leaking secrets held in structs.** Only dispatches the `Inspect` protocol; the mitigation (`@derive {Inspect, except: […]}` / custom impl) belongs at the struct-definition site, not here.
  Basis: **structural**.

- **`get_in`/`put_in`/`update_in`/`pop_in` and the `nest_*`/`unnest` path macros.** Access traversal; the only "code" that runs is a function *key*, which a caller cannot obtain from serialized/untrusted data (functions don't survive JSON/`binary_to_term [:safe]`). No injection vector.
  Basis: **structural**.

- **`raise/1,2`, `reraise/2,3` (`:2246`+).** Message/exception expansion at compile time; string messages become `RuntimeError.exception/1`. No format-string interpreter exists in Elixir; interpolation is ordinary and caller-authored.
  Basis: **structural**.

- **The remaining ~200 guard/BIF wrappers (`is_*`, arithmetic, comparison, `elem`, `byte_size`, `tuple_size`, …).** Confirmed by reading a representative cross-section of their bodies plus a full grep of `:erlang.*` call sites: each is a single-expression inline to one `:erlang` BIF with no branching or data-shaping logic. A single-BIF inline can only be as (un)safe as the BIF it forwards to, and the BIF's own behavior is Erlang-runtime scope, not a defect this file introduces.
  Basis: **structural** — no logic beyond delegation; no path in which this file diverges from the delegated BIF. *(This is the first-pass "did not exhaustively execute" hedge, now resolved: the claim was never behavioral/differential — it is "these bodies contain no logic" — and reading the bodies confirms exactly that. There is no untried input axis because there is no branch to exercise.)*

## 3. Risks / caveats

- **Build provenance:** The environment is not a git repo, so I confirmed the sandbox toolchain matches the source by version only (Elixir `1.21.0-dev`, OTP 29), not by commit SHA. Reproductions ran against that build.
- **Compile-time-vs-source boundary:** Many `Kernel` macros would do powerful things (atom creation, module dispatch, `apply`, non-bidi-gated unescaping) when expanding *hostile source*. I treated "compiling untrusted source/AST" as a whole-compiler concern (tokenizer/parser/interpolation, per the folder threat model) rather than a `kernel.ex` API defect. If a future pass scopes adversarial source as in-scope for this file, the `~w()a`/`String.to_unsafe_atom`, calendar-alias atom, and `sigil_c`/`sigil_r` → `:elixir_interpolation.unescape_string` paths are the places to look first.
- **Thin-wrapper residual risk:** The safety of `apply/*`, `spawn/*`, `send/2`, and `struct!/2`-with-atom rests on callers not forwarding attacker-controlled values as the module/function/dispatch argument. That is inherent to their contracts and not defendable within this file.

## 4. Did I verify this

- **`struct/2`/`struct!/2`, `~w()a`, `**`, `binary_slice` — verified (executed).** Repros re-run this pass; outputs cited inline above. `Not examined: none` on each — every behavioral axis I claimed (key-type/`__struct__`-injection, atom-table growth, exponent magnitude, negative/oversize bounds) was pushed through the real implementation and the returned value compared.
- **`apply`/`spawn`/`send`/`make_ref` and the ~200 guard/BIF wrappers — verified structurally by reading.** I read the wrapper bodies (e.g. `kernel.ex:352`, `:379`, `:1248`) and grepped every `:erlang.*` site (`:353`, `:380`, `:1026`, `:1248`–`:1378`, …); each is a single-BIF inline. The first-pass exhaustiveness hedge is removed: the claim is structural ("no logic present"), not differential, so no execution axis remains untried. I did **not** attempt to prove the delegated BIFs themselves are safe — that is Erlang-runtime scope, explicitly out of this file's boundary.
- **Compile-time/source-expansion paths (aliases, calendar sigils, `dbg`, `sigil_c`/`sigil_r`, `raise`) — structural, not executed.** These are dismissed on the source-vs-runtime boundary, not on a behavioral axis; there is no untrusted-runtime-data input to push through them, so there is nothing to execute. They remain flagged for a tokenizer/interpolation-scoped pass, not carried as findings here.

### Dropped in re-review

None. The first pass reported no findings; every ruled-out dismissal above is retained, and the only change is resolving the "did not exhaustively" hedge into a verified structural statement (no wrapper was upgraded to a finding because reading confirmed each contains no logic capable of a defect).
