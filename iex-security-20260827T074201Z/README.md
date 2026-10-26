# Security Scan Summary — `elixir/lib/iex` (IEx)

**Scan:** `iex-security-20260827T074201Z` · 13 source files · Elixir 1.21.0-dev / OTP 29

**Orientation.** IEx is a REPL whose *documented purpose* is arbitrary code execution, so "user runs code in the shell" is out of scope by design; the real defended boundary is who can reach the shell/distribution port. Against that backdrop, almost every surviving finding shares one attacker model: **a developer introspecting, prying into, or `open`-ing a hostile module or a forged `.beam` — e.g. shipped by a malicious dependency.** One of those chains is a genuine RCE. No `VALIDATION.md` was produced, so severities/confidences below are taken directly from the reviewer reports. This folder is *not* RFC-governed (`RFC_OBLIGATIONS.md`: empty standards list), so there are no conformance findings.

---

## 1. HIGH — Editor OS command injection → RCE (`introspection.ex`)
[`CSCAN-introspection-editor-os-cmd-injection`](./lib-iex-introspection.ex.md#cscan-introspection-editor-os-cmd-injection) · **High / high confidence · verified end-to-end**

`IEx.Introspection.open_location/2` (`introspection.ex:172`, reached from `open/1`) builds an editor command by string interpolation and escapes the file path with **`inspect/1`** — which is Elixir string-quoting, **not shell-quoting**. It then runs the string through `:os.cmd/1` → `/bin/sh -c`, so backticks and `$()` inside the path fire as command substitution. Crucially the path is **not caller-typed only**: `open(module)` derives it from `module.module_info(:compile)[:source]`, the compile-time source path baked into the `.beam`. A dependency shipping a source file legally named `` evil`cmd`.ex `` / `evil$(cmd).ex`, or a forged BEAM with a crafted `:source` attribute plus a matching on-disk file, turns the routine `open(SomeModule)` into **arbitrary OS command execution on the developer's machine** — no execution of the target module's own code required, only `EDITOR`/`ELIXIR_EDITOR` set (near-universal).

Reproduced on disk both directly (`open_location/2`, marker file created) and end-to-end through `open(EvilMod)` via a forged compiled `:source` path, on both the default and the `__FILE__`/`__LINE__` template branches. **Fix:** invoke the editor via `System.cmd/3` (argv, no shell), or POSIX single-quote the path — never `inspect/1`.

---

## 2. MEDIUM — Destructive code-unload from an untrusted message (`mix_listener.ex`)
[`CSCAN-mix_listener-untrusted-code-purge`](./lib-iex-mix_listener.ex.md#cscan-mix_listener-untrusted-code-purge) · **Medium / medium confidence · sink verified, reachability asserted**

`IEx.MixListener.handle_info({:modules_compiled, info}, …)` (`:45-63`) treats the whole `info` map as authoritative; its only gate is `info.os_pid == System.pid()`, a plain string compare an attacker sets to any value. It then feeds `info.modules_diff.changed/.removed` into `purge_modules/1`, which runs `:code.purge` + `:code.delete` on every named module. **Verified by execution:** an off-`os_pid` message unloads arbitrary in-memory modules (→ `UndefinedFunctionError`), and a double-purge kills any process running a targeted module — a repeatable DoS. Not RCE (payload is a non-executable term).

The load-bearing premise — that an *untrusted / cross-user* peer can deliver this message — lives in the `Mix.Sync.PubSub`/`Mix.PubSub` transport, which is **absent from this checkout**, so it is *asserted, not verified* (this is why it is Medium, not High, and appears as an **Unresolved** row in the cross-file ledger). Exposure is developer/shared-build-host only: `IEx.App` does **not** supervise `MixListener` (it's started by `iex -S mix`).

- **Companion (Low / high confidence · verified):** [`CSCAN-mix_listener-malformed-diff-crash`](./lib-iex-mix_listener.ex.md#cscan-mix_listener-malformed-diff-crash) — a malformed `modules_diff` (missing `:changed`/`:removed`, non-map, missing `:os_pid`) raises `MatchError`/`KeyError` past the catch-all and kills the GenServer at will. Escalation past `:one_for_one` restart intensity is asserted (supervisor absent).

**Fix (in-file defense-in-depth):** drop any module whose `:code.which/1` path is outside the project's build path before purging; durable fix is authenticating the PubSub transport and decoding with `[:safe]`.

---

## 3. LOW — Pry-consent bypass during shell boot (cross-file: `server.ex` + `pry.ex` + `broker.ex`)
[`CSCAN-cluster-shell-boot-pry-consent-bypass`](./CLUSTER_lib-iex-1.md#cscan-cluster-shell-boot-pry-consent-bypass) (CLUSTER_lib-iex-1) · **Low / high confidence** · *contradicts a per-file "ruled out"*

The consent gate for a *foreign* pry (`Allow? [Yn]`) lives only in `server.ex take_over?/6`, and steady-state `handle_common` correctly routes non-self evaluators there. But the boot path `shell_loop/3` (`server.ex:74-83`) handles the identical `{:take_over,…}` message by responding `take_over?(_, _, 1, true)` at **line 79** — the unconditional-accept variant, with **no evaluator-identity check and no prompt**. The initiator is not privileged: `IEx.Pry.pry/2` is a *public* function any process may call; `IEx.Broker` fans the genuine (un-forged) message to every server including the booting `shell_loop`, which accepts with no prompt and thus wins the accept race. Effect: an unrelated process forces an interactive pry session onto the operator's console for a foreign evaluator's binding — without the consent the steady-state path requires.

This directly refutes the `server.ex` per-file report, which dismissed line 79 by arguing a foreign takeover "requires forging a `:take_over` message with a matching `:evaluator` pid" — true for `handle_common`, **false for `shell_loop`**, which does no such check. **Fix:** route non-self evaluators in `shell_loop` through the prompting `take_over?/6`.

---

## 4. LOW — Terminal-escape injection from module-source bytes (three related file pairs)
**Low · CWE-150/116** — same class (source-derived bytes reach a terminal write unstripped), surfaced independently in three places, including a **Contradicted** ledger row:

- **[`CSCAN-cluster-info-source-path-terminal-escape`](./CLUSTER_lib-iex-0.md#cscan-cluster-info-source-path-terminal-escape) (CLUSTER_lib-iex-0, high confidence).** `info.ex` returns the `"Source"`/`"Module bytecode"` fields as **raw `Path.relative_to_cwd/1` strings** — un-`inspect`-wrapped, unlike every sibling field — and `helpers.ex i/1` (`:580`) prints them raw via `IO.puts(IEx.color(:eval_info, …))`. `i(HostileModule)` on a module whose compiled-in `:source` path carries ANSI/C0 escapes renders them to the terminal. The `info.ex` per-file reviewer concluded "everything reaches output through `inspect`" and never inspected the `i/1` consumer — only reading producer + consumer together reveals it.
- **[`CSCAN-crossfile-source-derived-terminal-escape-passthrough`](./CROSS_FILE.md#cscan-crossfile-source-derived-terminal-escape-passthrough) (CROSS_FILE, high confidence — Contradicted ledger row).** Prying into a module whose on-disk source contains escapes causes `pry.ex`'s raw `whereami` chardata to be printed unescaped by `server.ex:227`. `server.ex` dismissed this as "local trust"; `pry.ex` deferred the sink to `server.ex`; but `introspection.ex` *verified* those exact source bytes carry escapes to a terminal write. The two reviews classify the **same byte class oppositely** — the ledger flags this as **Contradicted** and recommends aligning to the finding.
- **[`CSCAN-introspection-terminal-escape-passthrough`](./lib-iex-introspection.ex.md#cscan-introspection-terminal-escape-passthrough--upgraded-from-hedged-assumption-in-first-pass) (introspection.ex, low confidence · verified).** In-file `IO.puts` sinks in `print_source/2`, `b/1`, `t/1`, `print_typespec/1` emit untrusted-BEAM-derived source paths and typespec names with no C0/ESC stripping (`Path.relative_to_cwd` and the `Code.quoted_to_algebra`/`Inspect.Algebra.format` pipeline both preserve ESC/BEL).

**Fix:** one IO-layer control-character scrubber (strip C0 except `\t`/`\n`, plus CSI/OSC) — but note the reports caution that scrubbing one path does *not* cover the others, so all sinks (`i/1`, pry-via-`server`, and the introspection printers) must be routed through it.

---

## 5. LOW — Quadratic typespec formatting DoS (`introspection.ex`)
[`CSCAN-introspection-typespec-algebra-quadratic-dos`](./lib-iex-introspection.ex.md#cscan-introspection-typespec-algebra-quadratic-dos--upgraded-from-hedged-assumption-in-first-pass) · **Low / low confidence · verified (timings)**

`format_typespec/3` runs `Code.quoted_to_algebra |> Inspect.Algebra.format`, which is **O(depth²)** (measured 9.6 → 34 → 131 → 564 ms as nesting depth doubles — each doubling ≈ 4× time). A forged BEAM carrying a pathologically nested spec/type makes `h(Mod.fun)`, `t(Mod)`, or `b(Mod)` burn seconds-to-minutes of CPU per call, hanging the session. Unlike the general `Inspect` case there is no `limit` cap on the document. This is the `Inspect.Algebra.format` quadratic tracked in `[[inspect-algebra-review]]`; file-local fix is to bound typespec nesting/size before formatting.

---

## 6. LOW — Unbounded atom creation via forged BEAM callbacks (`introspection.ex`)
`String.to_unsafe_atom` in `translate_callback_name_arity` (`:726`) · **Low / low confidence · reachability read from call chain, NOT executed** · CWE-770

`get_callback_docs/2` maps `translate_callback/1` over *every* entry from `Typespec.fetch_callbacks/1` **before** any filtering, and `translate_callback_name_arity/1` calls `String.to_unsafe_atom(macro_name)` unconditionally — bypassing the existing-atom check. A forged BEAM whose `callbacks` chunk enumerates arbitrarily many distinct `MACRO-<name>` entries mints a new atom per name when the developer runs `b`/`h`/`t` on the hostile module. The lowest-confidence item here: the forged-BEAM PoC was not built end-to-end.

---

## Cross-file ledger — items to note
- **Contradicted (1):** `pry.ex → server.ex` terminal-write trust classification — resolved into finding **#4** above (server.ex's "local trust" is contradicted by introspection.ex's verified result).
- **Unresolved coverage gaps (not findings, but gate real risk):**
  - **`mix_listener.ex` transport & supervisor** (×2) — `Mix.Sync.PubSub`/`Mix.PubSub`/`ListenerSupervisor` absent from checkout; the "untrusted sender" and "restart→subsystem-disable" halves of finding **#2** are unverifiable here. A reviewer with the Mix tree should confirm senders are unauthenticated/cross-user and the decode omits `[:safe]`. (One restored ruled-out note asserts `Mix.Sync.PubSub.decode_data/1` calls `non_executable_binary_to_term/1` without `[:safe]` — verify in the Mix tree.)
  - **`history.ex` → callers**, **`info.ex` `i/1` exclusivity**, **`helpers.ex` siblings**, **`mix.exs` → `config.ex` value consumption** — consuming-side validation not checked this pass; all are coverage gaps, not defects.

## Files reviewed with no surviving findings
`autocomplete.ex`, `broker.ex`, `config.ex`, `evaluator.ex`, `helpers.ex`, `history.ex`, `iex.ex`, `app.ex`, `mix.exs`, and — in isolation — `server.ex`, `pry.ex`, `info.ex` (each clean per-file, but the latter three each contribute to the cross-file findings #3 and #4 above). Recurring cleanly-dismissed patterns: evaluator input and `.iex.exs`/`import_file` are trusted-by-design (the feature); `remsh`/`erpc` sit behind the distribution-cookie full-trust boundary; autocomplete atom-minting and `:public` ETS tables are operator/co-tenant-scoped (attacker would already have in-VM RCE).
