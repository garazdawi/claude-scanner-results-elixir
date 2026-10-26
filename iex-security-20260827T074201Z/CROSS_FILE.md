# Cross-file findings

The IEx per-file reports use no literal `* Depends on:` lines; every cross-file dependency is expressed as prose hedges ("outside this file", "provenance in neighbour", "not in this checkout", "safety depends on callers"). I enumerated each such dependency and resolved it below.

## Asserted-dependency ledger

| Source file | Depends on | Verdict | Rationale (≤1 line) |
|---|---|---|---|
| autocomplete.ex | `server.ex:353-358,135` — `expand_fun` wired to reversed terminal buffer only | Confirmed | server.ex neighbour read; sole feed is the TAB line buffer, no network/disk caller |
| autocomplete.ex | `helpers.ex:1079-1080` — `exports/1` is operator-invoked helper, no untrusted feed | Confirmed | Read directly; `def exports(module \\ Kernel)` documented helper |
| autocomplete.ex | `broker.ex:37-42` — `evaluator/1` MatchError bounded to dead/stale shell pids | Confirmed | broker.ex caveat confirms MatchError is dead-pid-race, robustness-only |
| autocomplete.ex | `remsh/1` distribution cluster = full trust (shared cookie) | Confirmed | Threat-model given; broker.ex concurs |
| broker.ex | distribution cluster membership = fully trusted for `erpc`/`Node.list` | Confirmed | Threat-model given (co-node w/ cookie already omnipotent) |
| config.ex | `IEx.configure/1` (iex.ex) — values are validated developer input, not attacker data | Confirmed | iex.ex report verifies `configure/1` delegation + `color/2` |
| evaluator.ex | co-tenant BEAM trust — forged `{:eval,^server,…}` requires knowing pids | Confirmed | Threat-model out-of-scope; pins sender in every clause |
| history.ex | `evaluator.ex`/`helpers.ex` continue to feed only session-local results + integer indices | Unresolved | Neither caller report describes what it feeds History; nobody checked |
| info.ex | `IEx.Helpers.i/1` is the sole driver; `info/1` never auto-invoked on eval results | Unresolved | info.ex's own assumption; helpers.ex report doesn't confirm i/1 exclusivity |
| mix_listener.ex | `Mix.Sync.PubSub`/`Mix.PubSub` transport authenticates senders & decodes with `[:safe]` | Unresolved | Transport absent from checkout; could not read or re-run |
| mix_listener.ex | `Mix.PubSub.ListenerSupervisor` restart intensity (crash → subsystem disable) | Unresolved | Supervisor absent from checkout |
| mix_listener.ex | `config.ex:100-102,208` — `auto_reload?` is a validated boolean app-env read | Confirmed | Read config.ex neighbour |
| mix_listener.ex | `app.ex:15` — `MixListener` not supervised by IEx.App | Confirmed | Read app.ex; children are Config/Broker/Pry only |
| pry.ex | `server.ex` `take_over?` — pry-consent gating lives there | Confirmed | server.ex ruled-out entry: consent fail-closed (executed) |
| pry.ex | `broker.ex` — session-takeover authorization | Confirmed | broker.ex: accept requires unguessable ref, single-shot |
| pry.ex | `server.ex` — terminal write of `location`/`whereami` source bytes (escape handling) | **Contradicted** | server.ex dismisses as local-trust; introspection.ex verifies source-derived bytes DO carry escapes to terminal (see finding) |
| pry.ex | `backend.debug_info/4` dispatch requires prior code execution to reach | Confirmed | Definitional (loaded module already ran); in-file reasoning |
| server.ex | `evaluator.ex:272-318` — `:dot_iex` read-and-eval sink lives downstream | Confirmed | Read evaluator.ex; server only `Keyword.take`s it through |
| server.ex | `pry.ex:49-65`/`broker.ex:101-111` — `take_location`/`take_whereami` provenance | Confirmed | Read both; built from `Macro.Env`/source lines, forwarded verbatim |
| server.ex | `broker.ex:67-73` — `respond/4` trusts the consent boolean it is handed | Confirmed | Read broker.ex; enforcement genuinely lives in server.ex |
| iex.ex | `config.ex:77-90` — `IEx.Config.color/1` returns developer-config value, never untrusted | Confirmed | Read config.ex neighbour |
| iex.ex | `pry.ex` — `break!/4` defdelegate behaviour | Confirmed | Read pry.ex neighbour |
| helpers.ex | `IEx.Introspection`/`Evaluator`/`Broker`/`Server` siblings — explicitly NOT asserted safe | Unresolved | Definitional out-of-scope; safety not verified here (by design) |
| helpers.ex | distribution-peer trust for `:erpc`/`nl` | Confirmed | Threat-model given |
| mix.exs | `config.ex` — expansion of env defaults (prompt tokens, `history_size`, `inspect`) | Unresolved | Out of scope; consuming-side validation not checked in this pass |

## Findings

#### `CSCAN-crossfile-source-derived-terminal-escape-passthrough`
- **Finding ID** — `CSCAN-crossfile-source-derived-terminal-escape-passthrough`
- **Severity** — Low
- **Confidence** — high
- **Files involved** — `lib-iex-server.ex.md`, `lib-iex-pry.ex.md`, `lib-iex-introspection.ex.md`
- **Issue** — `server.ex` rules out terminal-escape injection at `IO.puts("Break reached: #{take_location}#{take_whereami}")` (server.ex:77,227) by asserting the provenance is safe: `take_whereami` is "built from … source lines (local trust), not attacker-supplied" (server.ex report, "Ruled out", line 29). `pry.ex` in turn declines its own terminal-escape entry by deferring the write sink to server.ex: `whereami_lines` "returns raw source bytes unescaped … the write-to-terminal sink is in `server.ex` (out of scope)" (pry.ex report, line 21), while confirming `\e[31m` in source survives into the returned chardata. But `introspection.ex` *verified* (Low) that source-file-derived bytes are exactly an attacker-controllable escape vector: `Path.relative_to_cwd/1` and the source-rendering pipeline "preserve ESC/BEL bytes into the in-file `IO.puts` sinks" for a hostile module (`CSCAN-introspection-terminal-escape-passthrough`). Chaining these: a developer who prys (`break!`/step) into a module whose on-disk source carries ANSI/C0 escapes causes `pry.ex`'s raw `whereami` chardata to be printed unescaped by `server.ex:227` — the "local trust" dismissal in server.ex is contradicted by introspection.ex's verified result that source bytes reach a terminal write unstripped. The two reviews classify the *same byte class* (module source lines) oppositely: trusted in server.ex, attacker-influenced-and-real in introspection.ex.
- **Resolution** — Read the actual `server.ex` `IO.puts` at lines 77/227 and confirm `take_whereami` is emitted without a control-byte scrubber, then align the trust classification: either upgrade server.ex's ruled-out entry to a Low finding matching introspection.ex, or apply the single IO-layer control-character scrubber that introspection.ex's suggested fix already proposes (strip C0 except `\t`/`\n`, and CSI/OSC) so all three sinks (introspection `IO.puts`, pry-via-server `whereami`) are covered once. Severity stays Low: self-inflicted (developer must pry into / introspect a hostile module), single-session.

## Notes on the Unresolved rows

The load-bearing `mix_listener.ex` reachability premise (`Mix.Sync.PubSub` transport authenticates senders / decodes with `[:safe]`) is genuinely unverifiable in this checkout — the transport and its supervisor are absent. Per the ledger rule this stays an Unresolved coverage gap, **not** a minted finding: mix_listener.ex already carries its own in-file `CSCAN-mix_listener-*` findings for the destructive endpoint, and the missing half (untrusted sender) cannot be named at a second file:line here. A reviewer with the Mix tree should confirm (i) senders are unauthenticated / cross-user reachable and (ii) the decode omits `[:safe]`; both `history.ex`→callers and `mix.exs`→`config.ex` value-consumption rows are likewise coverage gaps, not defects.
