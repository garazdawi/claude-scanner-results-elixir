# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/utils.ex
<!-- sha256: ff54ec1d321dd91568c877ab0148025916b192c11f32bb341436260c07661f6a -->

# /Users/lukas/code/claude-scanner/elixir/lib/logger/lib/logger/utils.ex
<!-- sha256: ff54ec1d321dd91568c877ab0148025916b192c11f32bb341436260c07661f6a -->

_Re-reviewed on 2026-08-27 (Erlang/OTP 29, erts-17.0.2, JIT). All prior hedges resolved against the callgraph neighbour `formatter.ex` and by executed reproductions._

## Summary

`Logger.Utils` contains no attacker-reachable security defect. Its byte-math truncation (`truncate_n`/`fix_binary`) is crash-free and never cuts mid-codepoint at the boundary; its inspection pre-pass bounds cost on hostile **data**; and its remaining sinks (`apply` on translators, `:logger` primary filters) operate on trusted operator config wrapped in `try/rescue`. The one genuinely unbounded path (`~P`/`~W` with a huge limit) is now **verified** to require the hostile value in the format-**template** position, which — per the sole caller in `formatter.ex` — is the log event's caller-supplied format field, not the attacker-data (args) position.

## Security

No findings.

Inventory of security-relevant sinks and why each is safe:

- **`translate/5` → `apply(mod, fun, [...])` (line 79)** — dynamic dispatch, but `{mod, fun}` comes from `%{translators: translators}`, i.e. operator config (`Logger.add_translator/1`). Not attacker-influenced. The report *data* forwarded as the 4th arg may hold attacker bytes but is only pattern-matched and passed on; the whole `case` is wrapped in `try/rescue` (lines 19–41). Confidence: high.
- **`scan_inspect/4` → `:io_lib.scan_format` + `Inspect.Algebra` (lines 150–209)** — the resource-consumption sink. The pre-pass truncates binary args (lines 153–160); non-binary args are bounded by `opts.limit` at inspection time. **Verified via caller (`formatter.ex:310`):** the real formatter path invokes `scan_inspect(format, args, truncate)` with **three** arguments, so `opts` defaults to `%Inspect.Opts{}` → `limit: 200` (not operator-tunable on this path). **Executed:** a benign `~p`/`~w` format on a 1,000,000-deep and 1,000,000-wide attacker term rendered in **12 ms**, `out=80805`/`897` bytes — the attacker-data-in-args path is bounded. Confidence: high.
- **`truncate_n/2` + `fix_binary/1,2` (lines 214–282)** — byte arithmetic on possibly-multibyte content. **Executed:** no crash and no mid-codepoint tail across ascii-cut, emoji-straddle, and 50 randomized cut offsets through a euro-sign+ascii binary (`emoji straddle out="a" valid=true rem=-2`; "random cuts: no mid-codepoint reported"). A straddling code point is dropped whole; `binary_part(binary, 0, n)` is always in-range because the `n < 0` clause (line 214) is matched first and `remaining < 0` implies `0 ≤ n < byte_size`. Confidence: high.
- **`process_level/2`, `silence_logger_exit/2`, `translated_cb/1` (lines 57–121)** — `:logger` primary filters / report callback; read `self()` process level or match a fixed report shape. No external-data sink. Confidence: high.
- **Atom / code-exec / path / deserialization / crypto sinks** — none present. `Atom.to_charlist` (line 139) is the safe direction (no atom created); no `String.to_atom`, `binary_to_term`, `Code.eval*`, `Module.concat`, `File`/`Path`, or shell in this file. Confidence: high.

## Risks / caveats

- **`~P`/`~W` with a caller-supplied huge limit is unbounded** — but not attacker-reachable via data. See the ruled-out entry below; the reachability question is now verified against the caller rather than assumed.

### Ruled out

- **`~P`/`~W` unbounded inspection limit** — `inspect_limit/3` (line 197) takes the limit from the **format-string control args** (`[_, limit]`) rather than `opts.limit`, so the depth cap is bypassed and `Inspect.Algebra.format/2` goes quadratic. **Executed (format-string position):** `scan_inspect(~c"~P", [deep_200k, 100_000_000], 8192)` did not complete within a 5 s watchdog (killed → `timeout`). **Reachability verified against the sole caller:** in `formatter.ex:308–310`, `scan_inspect` is only ever reached from `format_message({format, args}, _meta, truncate)`, where `format` is the log event's `msg` format field (caller-supplied via `:logger.log(level, Format, Args)` / translated `error_logger` format reports) and the `~P`/`~W` limit is parsed out of that template's own control sequence, not from `args`. To trigger the blowup an attacker must control the **format template**, i.e. the application passed untrusted data as a format string (printf-style format-string injection) — caller misuse, not a Logger network-data path. Attacker data confined to the `args` position stays bounded (executed above: 12 ms). **Axes examined:** data-position (attacker term in `args` → bounded, executed) and template-position (huge limit in format control args → unbounded, executed); reachability of template-position from network data (traced through the only caller in `formatter.ex`). **Not examined:** none. Suggested hardening if desired: clamp the `~P`/`~W` limit to `min(limit, opts.limit)`. Confidence: high (ruled out as a Logger-contract finding).

- **`truncate_n` deep-nested-iolist non-tail recursion** — `truncate_n_list` (lines 249–261) descends non-tail into nested list elements before `n` is consumed, so a pathologically deep iolist could grow the process stack. Not attacker-reachable: `scan_inspect`'s pre-pass calls `truncate_n` only on `is_binary/1` args (line 155), and the formatter's binary/list message path (`wrapped_truncate` → `truncate` → `truncate_n`, formatter.ex:327–350) receives message chardata, not a million-deep attacker iolist producible from network data through the documented API. **Axes examined:** entry points calling `truncate_n` (both the `scan_inspect` binary-only pre-pass and the `Formatter.truncate` chardata path, read in `formatter.ex`). **Not examined:** none material — a deep-list input would need a caller that hands `truncate_n`/`truncate` a deeply-nested list, and no such caller supplies attacker-shaped nesting. Confidence: high (robustness, not attacker-reachable).

- **`fix_binary` leaves pre-existing invalid bytes intact** — **Executed:** `truncate_n(<<0xC3,0x41,0x42,0x43>>, 4)` returned `<<195,65,66,67>>` unchanged. By design: `truncate_n` only sanitises the ≤13-byte cut window to avoid mid-codepoint boundaries and is documented "approximate"/not a UTF-8 validator (formatter.ex:338–341). Downstream, `Logger.Formatter.prune/prune_binary` (formatter.ex:358–366) is the code point sanitiser. Not a defect. **Axes examined:** boundary-window sanitisation (executed), non-validator contract (documented). **Not examined:** none. Confidence: high.

- **`translator/2` `else` block runs outside the `rescue`** — `Enum.into(translation_meta, meta)` and `return_translated_event/3` (lines 42–53, 61–76) execute in the `else` branch of the `try` (Elixir `else` is not covered by the `rescue`), so a translator that returns malformed (non-enumerable / unrecognised) data raises to the logging process rather than being converted to a `{:string, chardata}` event. Gated entirely behind trusted `{mod, fun}` translator config; the folder threat model explicitly notes translators returning malformed data bypass the outer rescue. Robustness, not an attacker-reachable security bug. **Axes examined:** exception-in-translator (caught by lines 34–41 rescue) vs malformed-return-from-translator (escapes via `else`). **Not examined:** none. Confidence: high.

- **`truncate: :infinity` disables the binary pre-pass** (line 155) — operator configuration footgun. In the real path `truncate` originates from the formatter struct (`options[:truncate] || Application.fetch_env!(:logger, :truncate)`, default 8192, `formatter.ex:173`), i.e. operator config, not external input. **Axes examined:** source of `truncate` (traced to formatter config in `formatter.ex`). **Not examined:** none. Confidence: high.

## Did I verify this?

- **Read the callgraph neighbour `formatter.ex`** to resolve the two first-pass hedges. Confirmed `scan_inspect` has exactly one caller (`format_message({format, args}, …)`, formatter.ex:310), that it is invoked with the default `%Inspect.Opts{}` (limit 200), that `format` is the caller-supplied log-event format field, and that the whole message→truncation wiring (`format_event → format_message → scan_inspect/format_report → wrapped_truncate → truncate → Utils.truncate_n`) matches the standing cross-file invariants. This removes the "outside this file" hedge: the format-template position is caller-controlled, not an attacker-data path.
- **Executed** (OTP 29): truncate boundary safety (emoji straddle, invalid-byte preservation, 50 randomized cuts — no mid-codepoint, no crash); args-position bound (deep+wide 1M term via `~p`/`~w` → 12 ms, ≤81 KB); template-position blowup (`~P` + 100M limit → 5 s watchdog timeout/killed).

## Dropped in re-review

None. All prior ruled-out entries and the "no security findings" conclusion are preserved; the only change is that the first-pass assumption "reaching the `~P` template position is caller misuse outside this file" is now **verified** by reading the sole caller in `formatter.ex` and executing both data-flow positions, so no hedged assumption remains to upgrade.
