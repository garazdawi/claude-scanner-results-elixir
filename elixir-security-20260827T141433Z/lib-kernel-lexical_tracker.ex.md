# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/lexical_tracker.ex
<!-- sha256: 3dd001df35a57c7af964509ddd0aae13f31a3291d2ff21b595856f6eee7a9075 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/lexical_tracker.ex
<!-- sha256: 3dd001df35a57c7af964509ddd0aae13f31a3291d2ff21b595856f6eee7a9075 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

Confirmed clean. Re-review discharged every hedge from the first pass by reading the neighbour modules named in the standing context and by executing the one differential axis (`put_in` on a nil map value). The pid-never-escapes claim, the write/read-cache ordering claim, and the `warn_import` nil-crash reachability claim are now backed by cited neighbour code and one executed check rather than by assumption.

## Summary
`Kernel.LexicalTracker` is a per-compilation, `@moduledoc false` `gen_server` that accumulates in-memory bookkeeping (module references, imports, aliases, requires, a compile-env ordset, and an AST cache). It has no code-execution, atom-creation, file, deserialization, crypto, or dynamic-dispatch sinks. Its pid is created by `elixir_lexical:run/3` (`?tracker:start_link()`, `elixir_lexical.erl:14`), stored only in `env.lexical_tracker`, `unlink`ed and `stop`ped in the same `after` block (`elixir_lexical.erl:27-28`), and is never handed across a trust boundary. Every `module` argument arrives as an already-interned atom emitted by the tracer (`elixir_lexical.erl:35-73`) or the expander (`elixir_expand.erl`), used solely as a map key — never re-atomized here.

## Security
No findings.

### Ruled out

- **Atom-table exhaustion (CWE-770).** No `String.to_atom`/`binary_to_atom`/`List.to_atom`/`Module.concat` in the file; `module` args arrive as atoms (most guarded `when is_atom/1`) and are used only as map keys. *Confidence: high. Basis: verified — full read of all 318 lines; grep for atom-creation calls returned no matches. Axes examined: input type (atom vs binary/charlist) | Not examined: none — every public/`@doc false` entry point takes `module` already-atomized and the file contains no `to_atom`-family sink to feed.*

- **Code execution / eval / dynamic dispatch (CWE-94).** No `Code.*`, `apply/3`, `:erlang.apply`, `eval`, or `spawn`. `write_cache`/`read_cache` stash a `defmodule` body AST under an `:erlang.unique_integer()` key and read it back later. *Confidence: high. Basis: verified — grep for exec sinks, no matches; traced the sole caller (kernel.ex:5186-5188) — see the cache-ordering entry below.*

- **Deserialization (CWE-502).** No `:erlang.binary_to_term`. *Confidence: high. Basis: verified — grep, no matches.*

- **File / path / command / network sinks.** `set_file`/`reset_file` only store/clear a path string in `state.file`; no `File.*`, `Path.*`, `:os.cmd`, `System.cmd/shell`, or outbound request. The stored file is only later read back by `elixir_lexical:with_file/3` (`elixir_lexical.erl:82-90`) and threaded into warning `Meta` — never used as a sink. *Confidence: high. Basis: verified — grep, no matches; read `handle_cast({:set_file,…})`/`:reset_file` and the neighbour `with_file/3`.*

- **Shared-state poisoning across callers.** State is not global (no ETS/`:persistent_term`/`Application.put_env`/`Process.put`); it is one `gen_server`'s heap, scoped to one compilation unit whose pid is created and destroyed inside a single `elixir_lexical:run/3` (`elixir_lexical.erl:14,27-28`). *Confidence: high. Basis: verified — grep for shared-state sinks, no matches; `init/1` state is a plain map; pid lifecycle read in the neighbour.*

- **write_cache → read_cache ordering / race.** kernel.ex:5187 calls `write_cache` (a `cast`) during `defmodule` expansion and gets back the integer key; kernel.ex:5188 emits `read_cache(pid, integer)` into a `quote` that runs strictly later when the generated module code executes. The write `cast` is therefore enqueued before the read `call` is ever performed, and the `gen_server` mailbox is FIFO, so the write is applied first. Key and pid are compiler-generated, not attacker-influenced. *Confidence: high. Basis: verified — read the caller at kernel.ex:5184-5192; the write precedes emission of the deferred read. This is an ordering claim over OTP mailbox + caller sequence, not a differential claim, so there is no divergence axis to push. Not examined: none applicable.*

- **Crash-on-unexpected-message DoS.** `handle_call`/`handle_cast` have no catch-all, so an unrecognized message would `function_clause`-crash the server, but only code holding the compiler-internal pid can send — verified the only senders are `elixir_lexical:trace/2` (`elixir_lexical.erl:35-73`), the `warn_*`/`collect_*` helpers there, and kernel.ex's `defmodule`. `handle_info/2` already ignores stray info messages. A crash would fault only that one compilation's tracker. *Confidence: high. Basis: verified — traced every sender via grep of `?tracker`/`Kernel.LexicalTracker`; no external route to the pid. Axes examined: message shape (known cast/call vs unknown), sender identity | Not examined: none — sender set is closed and enumerated.*

- **Unbounded memory growth.** `references`, `imports`, `aliases`, `requires`, `cache`, and `compile_env` grow monotonically with no eviction, but each is bounded by the number of distinct symbols/nested modules in the source being compiled — linear in input, no amplification. *Confidence: high. Basis: verified — reviewed every `handle_cast` mutation; each adds at most O(1) per distinct source symbol.*

- **`warn_import` nil-crash — `put_in(state.imports[module][@warn_key], true)` (line 277).** This *would* raise if `state.imports[module]` were absent: I executed `put_in(%{imports: %{}}.imports[Foo][0], true)` and it raised `could not put/update key 0 on a nil value`. However, the crash is unreachable in the real flow: at the sole call site the expander casts `?tracker:add_import(Pid, ERef, Only, Meta, false)` **immediately before** emitting the deferred `warn_import` call (`elixir_expand.erl:174-175`), so `state.imports[ERef]` is always populated before `warn_import` runs (mailbox FIFO: the `add_import` cast precedes the later-executed `warn_import` cast). A hand-built out-of-order call could crash, but only faults the caller's own compilation, and the pid is not attacker-reachable. *Confidence: high (was: excluded-per-scope assumption in first pass — now verified). Basis: verified — read the call site at `elixir_expand.erl:157-183`; ran the nil-`put_in` axis. Axes examined: value present vs nil at the `[module]` slot; call ordering add_import→warn_import | Not examined: none.*

## Did I verify this
- **Hedge 1 (first pass: "reasoned from OTP mailbox ordering; severity moot").** Upgraded to *verified* by reading kernel.ex:5184-5192: `write_cache` is a `cast` executed during expansion; `read_cache` is emitted into a `quote` executed later. Ordering holds; inputs are compiler-generated.
- **Hedge 2 (first pass: "scope boundary — inputs assumed well-formed atoms, out of scope").** Verified for what *this* file owns: `elixir_lexical.erl:35-73` and `elixir_expand.erl:90-178` show every `module`/`alias` argument reaches the tracker as an already-interned atom from the tracer/expander, used only as a map key here (no re-atomization). The narrower residual — whether the *upstream* tokenizer could mint a confusable/non-NFC atom — is genuinely a different file (`elixir_tokenizer.erl`) and is not a hedge about this file's behavior: any such atom is inert as a map key here. Not carried as a finding.
- **Hedge 3 (first pass: "excluded per scope — internal call-ordering contract").** Upgraded to *verified* by reading `elixir_expand.erl:174-175` (add_import precedes warn_import emission) and executing the `put_in`-on-nil axis to confirm both the crash shape and its unreachability. Remains ruled out, now with evidence.

## Dropped in re-review
None. All first-pass dismissals are preserved; three were upgraded from assumed/hedged to verified as noted above. No first-pass finding existed to drop, and no dismissal was weakened.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 2 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`put_in(state.imports[module][@warn_key], true)` crash on a missing module (`warn_import`, line 277).** Would raise if `warn_import` preceded `add_import` for that module — but that is an internal call-ordering contract enforced by the expander, not reachable via the documented API with an out-of-contract argument, and would at worst fault the caller's own compilation. *Basis: verified by reading the handler; excluded per scope (internal contract).*

- **write_cache→read_cache race / concurrency.** `write_cache` casts then returns the key; a same-process `read_cache` call is enqueued after the cast and the `gen_server` mailbox is FIFO, so the write is applied first. Not attacker-influenced regardless (pid/key are compiler-generated). *Basis: reasoned from OTP mailbox ordering; not attacker-reachable, so severity is moot.*
