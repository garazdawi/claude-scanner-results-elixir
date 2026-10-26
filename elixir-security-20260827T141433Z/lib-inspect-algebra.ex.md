# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/inspect/algebra.ex
<!-- sha256: 2c43d2ecba2b70adc62fc01b07406fc6f070090aa37107dc5fc10a209933ab7d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:11:26Z_

# Security Review — `lib/elixir/lib/inspect/algebra.ex`

## Summary
The pretty-printing engine's `fits?/4` fitting check is quadratic (O(N²)) on deeply-nested or break-free documents, giving an unbounded CPU-DoS reachable through `inspect/2` (with `pretty: true` and a raised/`:infinity` limit) and through the documented `Inspect.Algebra.format/2` API; no other meaningful security issues were found.

## Security

#### `CSCAN-algebra-fits-quadratic-dos`
- **Finding ID:** `CSCAN-algebra-fits-quadratic-dos`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `algebra.ex:1202` (`fits?/4`, the `when k > w and b?` short-circuit) together with the per-group callers at `algebra.ex:1352` and `algebra.ex:1360` (`format/4` for `doc_group`); root behaviour documented in the comment at `algebra.ex:1197-1201`.
- **Class:** Resource consumption (algorithmic-complexity DoS, CWE-407 / CWE-770).
- **Issue:** `format/4` calls `fits?/4` once per group to decide flat-vs-break. `fits?/4` only stops scanning early when the column exceeds the width **and** a break has already been seen: `defp fits?(w, k, b?, _) when k > w and b?, do: false`. For any subtree that contains no break yet (`b? == false`), the width bound `k > w` is ignored and the scan proceeds to the end of the remaining document. Because groups nest and each enclosing group re-runs `fits?` over content that contains further break-free groups, the total work is O(N²) in the number of nodes, even though the rendered output is only O(N) bytes. Node count is bounded by the global `:limit` budget (default `200`, so defaults are safe), but is unbounded when the caller passes `limit: :infinity` or a large limit — and nesting *depth* is never separately capped.
- **Impact / threat:** A process that pretty-prints untrusted, deeply-nested data with a raised limit — e.g. `inspect(user_data, pretty: true, limit: :infinity)` or `IO.inspect(user_data, pretty: true, limit: :infinity)` in a logger/debug path, or a library that builds algebra documents from untrusted structure and calls `Inspect.Algebra.format/2` — spends quadratic CPU. A ~625 KB nested term consumes ~55 s of single-core CPU, and the cost grows as depth²/2 with no ceiling, so a low-hundreds-of-KB input crosses into minutes. The trust boundary crossed is untrusted term/structure → CPU exhaustion of the inspecting/logging process (request-handler stall, log-pipeline stall).
- **Evidence:** `verified`.
  - Root cause (call counts) via `:cprof` on `inspect(nested_1tuple, pretty: true, limit: :infinity)`: for depth **N=500**, `Inspect.Algebra.fits?/4` was invoked **881,254** times (~3.5·N²) versus `format/4` only 3,506 times.
  - Quadratic wall-clock via `inspect/2` (`pretty: true, limit: :infinity`), output size stays linear (2·N bytes) while time quadruples per doubling:
    ```
    depth=10000 (~156KB term): 2.338 s
    depth=20000 (~312KB term): 11.602 s   (4.96x)
    depth=40000 (~625KB term): 55.143 s   (4.75x)
    ```
  - Same defect through the pure documented API — `Inspect.Algebra.format(group(group(...group(empty())...)), 80)` (break-free nested groups):
    ```
    groups=10000: 0.397 s
    groups=20000: 1.788 s   (4.5x)
    groups=40000: 11.012 s  (6.2x)
    ```
  - Control: `pretty: false` (which `Kernel.inspect/2` maps to width `:infinity`, `kernel.ex:2474-2478`) is linear (150→362→1151→2919 µs for N=1000→8000), because the group clause short-circuits on `w == :infinity` and never calls `fits?`. Default `inspect/2` (limit `200`) is constant-time (~27 µs, 405-byte output) regardless of depth.
- **Suggested fix:** Bound the `fits?/4` scan by the width irrespective of `b?`: once `k > w` the flat rendering already exceeds the line, so the group can be reported as not-fitting in O(width) rather than scanning to the document end. Adjusting the guard at `algebra.ex:1202` toward `when k > w` (dropping the `and b?`), or otherwise capping/short-circuiting the scan once the width is exceeded, makes each `fits?` O(width) and the whole render O(N·width). This changes the "a break-free document always fits" semantics documented at `algebra.ex:1197-1201`, so it must be validated against the code-formatter test suite; a defensive alternative is an explicit maximum nesting depth in the `Inspect` traversal so `limit: :infinity` cannot expand depth without bound.

### Ruled out
- **`Inspect.Opts.default_inspect_fun/1` (`algebra.ex:186-188`) writing a VM-global via `:persistent_term.put/2`.** This is a documented, deliberately global override whose own docs call it "extremely discouraged" for libraries; its stated purpose is redaction/privacy filtering (a security *feature*). Reaching it requires calling the setter (code execution), not untrusted data, so it is out of scope. Basis: verified by reading the function and its `@doc`; it takes a `fun` (guarded `is_function(fun, 2)`) and stores it, with no untrusted-data path into this file.
- **`to_doc_with_opts/2` invoking `fun.(struct, opts)` (`algebra.ex:401`, `algebra.ex:453`).** `fun` is `opts.inspect_fun`, a caller-supplied configuration value, not derived from the inspected term; dispatch to a struct's `Inspect` implementation is inherent to the protocol and lives in other files. No code-execution boundary is introduced here.
- **`valid_struct?/1` calling `module.__info__(:struct)` (`algebra.ex:456-465`).** `module` is the struct tag from `%module{}`, an atom that must already exist in the term (no atom creation). `__info__/1` is a pure generated function; non-Elixir modules raise `UndefinedFunctionError`, which is rescued to `false` (term then inspected as a map). Verified by reading the `try/rescue/else`; it is a conservative guard, not a sink.
- **Process-dictionary `:inspect_trap` flag (`algebra.ex:410-443`).** Process-local and transient; set only in the `else` branch that owns the balancing `after Process.delete/1`, and re-entrant inspects take the already-set branch without touching it. No cross-process or cross-eval leakage. Verified by tracing the `if Process.get/try/after` structure.
- **Verbatim emission of `doc_color`/`color/2` binaries and raw binary docs into output (`algebra.ex:792-794`, `algebra.ex:1294-1295`, `algebra.ex:1300-1301`) — ANSI/terminal-escape injection.** The color strings come from `Inspect.Opts.syntax_colors` (caller config) or explicit `color/2` calls, not from the inspected term; escaping of term string content is the responsibility of the individual `Inspect` implementations in other files. This module is the rendering engine and emits its inputs by design; no escaping boundary is owned here.
- **Non-tail recursion in `fold/2`/`concat/1` (`algebra.ex:1143-1147`, `algebra.ex:783-785`).** Recursion depth equals the list length passed by the library caller; `container_doc` bounds its list to ~`limit` items, and the BEAM grows the process stack dynamically (memory-bounded, no fixed overflow). Reaching a problematic depth requires the caller to hand `concat/1` a giant list directly — caller misuse, not untrusted-data flow into this file.
- **Standards/Unicode obligations from the folder checklist (BIDI, NFC, mixed-script, `\u` escapes, etc.).** None apply to this file. `Inspect.Algebra` performs no identifier tokenization, normalization, case-folding, or escape decoding; its only Unicode touch is `String.length/1` in `string/1` (`algebra.ex:735`) for grapheme-based width accounting, which makes no equality/security decision. The Unicode checks live in `elixir_tokenizer.erl`/`elixir_interpolation.erl`, not here.

## Risks / caveats
- **Reachability caveat on the finding:** the quadratic path is only exercised when pretty printing at a *finite* width (`pretty: true`) **and** the effective node budget is large — i.e. `limit: :infinity` or a substantially raised `:limit`. Default `inspect/2` (`limit: 200`) and `pretty: false` (width `:infinity`) are both safe, as verified above. Severity is set to Medium rather than High on that basis; a deployment that never raises the limit on untrusted input is not exposed. I did not survey callers across the wider codebase to quantify how often `limit: :infinity` meets untrusted data in practice.
- I did not attempt to produce a corrected `fits?/4` and re-run the formatter test suite, so the suggested guard change is a direction, not a validated patch — the `algebra.ex:1197-1201` comment indicates the "break-free always fits" behaviour is intentional and may have subtle formatter-visible consequences that need regression testing.
- Reproductions were run on Elixir 1.21.0-dev / OTP 29 built from this source tree; absolute timings are hardware-dependent, but the O(N²) scaling (≈4× per input doubling) is the load-bearing evidence and is consistent across both the `inspect/2` and pure-`format/2` reproductions.
- Review was scoped to this single file; cross-file behaviour (the per-type `Inspect` implementations that build the documents, and `Kernel.inspect/2`'s width selection) was consulted only enough to establish reachability of this file's public API.

