# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/examples/difference.exs
<!-- sha256: 8d3f4595f22e51d2c0664a62610630f7c8ee5c94ad939e93b6cd197867e4dac4 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/examples/difference.exs
<!-- sha256: 8d3f4595f22e51d2c0664a62610630f7c8ee5c94ad939e93b6cd197867e4dac4 -->

_Re-analysed on 2026-08-27_

I re-read the entire 196-line file to re-work each hedge in the prior report.

**Phase 1 — inventory (re-confirmed).** This is `difference.exs`, an ExUnit **example/demo script**. Its `@moduledoc` (lines 9–12) states it "contains failing tests to see difference highlighting in action," and the run comment (line 5) documents `make compile && bin/elixir lib/ex_unit/examples/difference.exs`. Sink-class sweep:

- **Code execution / compile-time:** `defmacrop one, do: 1` (line 19, returns literal `1`) and `use ExUnit.Case` (line 13). No `Code.eval_*`, `apply`, or dynamic dispatch.
- **Atom creation:** none — every atom (`:ok`, `:hello`, `:hex`, `:message`, keyword keys `file:`/`line:`/`port:`/…) is a source literal.
- **Command / file / path / archive:** none.
- **Deserialisation:** no `binary_to_term`.
- **Template / interpolation:** no `#{}` anywhere; the `"fox hops over \"the dog"` strings (line 31) and `~c"..."` charlists (lines 75–76) are plain literals.
- **Concurrency / shared state:** `send(self(), {:message, i})` (lines 168, 174, 180), `send(self(), 12)` (line 186), `self()`, `make_ref()` — all self-directed, bounded by fixed literal ranges (`1..2`, `1..11`).
- **Resource consumption:** `Enum.into(1..15, …)` (line 97), `Enum.reduce(5..10, …)` (line 98), `for i <- 1..11` (line 180) — fixed, tiny, compile-time-known bounds.
- **Everything else (network, crypto, unicode, validation, round-trip, protocol dispatch):** no call sites.

**Phase 2 — trace (re-confirmed).** Every value in the file is a hard-coded literal authored inline. There is **no external-input channel of any kind**: no function argument, no file read, no network, no env var, no message received from an untrusted process (the only `receive`-side constructs — `assert_received`, lines 164/170/176/182/188 — read from the process's own mailbox, fed only by the self-directed `send`s above, and every `assert_received` is deliberately unsatisfied). The file is an executable example, not a library module; nothing `import`s or calls `Difference`. The deliberately-failing `assert a == b` lines feed static literals into ExUnit's diff renderer so a developer can eyeball the highlighting output.

**Bearing on the hedges.** The prior report's hedges ("out of scope for this file," "I assumed") all concerned whether a bug in `ExUnit.Diff`/`ExUnit.Formatter` should be charged against this file. That question does not turn on how those modules behave — it turns on whether this file can *drive* them with untrusted data. Re-reading the whole file confirms the load-bearing fact directly: the pathological terms (`make_ref()`, `{}` vs `{true}`, improper list `[1 | "b"]`, mismatched struct arities) are **static literals in this file**, not values derived from any input. There is therefore no data-flow path by which an attacker reaches `ExUnit.Diff` *through* `difference.exs`. This is verifiable from `difference.exs` alone; reading `ExUnit.Diff` cannot change it, because the absence of an input channel — not the renderer's guard behaviour — is what makes the entry non-reachable. Accordingly I keep these as ruled-out (grounded, not assumed) rather than upgrading to low-confidence findings: an upgrade would assert a reachable sink that the file's contents affirmatively contradict.

No sink exists to drive, so there is no behavioural claim to reproduce and no cross-implementation axis to push a value through.

## Summary
`difference.exs` is a static ExUnit example script full of deliberately-failing assertions used to visually demonstrate the framework's diff-highlighting output. Re-reading all 196 lines confirms it takes no external input, exposes no public API, and contains no reachable dangerous sink. No findings.

## Security
No findings

### Ruled out

- **Diff-renderer crash on pathological terms** — Confidence: high (ruled out). The exotic values (`make_ref()`, `{}` vs `{true}`, improper list `[1 | "b"]`, reverse-ordered keyword lists, `%User{}` arity mismatch) are inputs to `ExUnit.Diff`/`ExUnit.Formatter`, but every one is a compile-time literal authored inside this file (lines 55–56, 68–72, 90–94, 122–123). There is no external-input channel in the file that could carry an attacker-chosen term to the renderer, so any renderer defect is not reachable *via this file*. Grounded in the full re-read, not in an assumption about the renderer.
  Axes examined: presence of an external-input channel (function arg / file / network / env / untrusted message); origin of every term passed to `assert` (all inline literals). | Not examined: none — the only untried axis would be the renderer's own behaviour on hostile terms, which is not an axis of *this* file and cannot be reached through it; it belongs to a review of `ExUnit.Diff`/`ExUnit.Formatter`.

- **Interpolation / template injection** — Confidence: high (ruled out). No `#{}` occurs anywhere; all strings (lines 31–32, 43–44) and charlists (lines 75–76) are literals, so no value reaches a downstream interpreter.
  Axes examined: every string/charlist literal for interpolation syntax. | Not examined: none.

- **Atom-table exhaustion** — Confidence: high (ruled out). No `String.to_atom`/`binary_to_atom`/`Module.concat`/`to_unsafe_atom`; every atom is a compile-time literal.
  Axes examined: all atom-producing call sites. | Not examined: none.

- **Resource consumption (message flood / large maps)** — Confidence: high (ruled out). The `send(self(), …)` loops (lines 168, 174, 180, 186) and `Enum` builders (lines 97–98) are bounded by fixed literal ranges (`1..2`, `1..11`, `1..15`, `5..10`), self-directed, and trivially small; none is attacker-influenced.
  Axes examined: every loop/builder bound (all literal ranges). | Not examined: none.

- **Compile-time execution on the build machine** — Confidence: high (ruled out). At load time the script only calls `ExUnit.start(seed: 0)` (line 6) and defines literal-returning macros/tests; it reads no file and hits no network, and it is run manually by the developer who already trusts it, not pulled from a dependency or archive.
  Axes examined: all load-time (module-body / macro) evaluation sites. | Not examined: none.

## Risks / caveats
- I reviewed only this file, which is correct scope: it contains exclusively hard-coded literals with no external-input channel, so nothing here is externally reachable. Whether `ExUnit.Diff`/`ExUnit.Formatter` mishandle exotic terms is a question for a review of *those* files; it is not reachable through `difference.exs` and is not an assumption load-bearing on any conclusion here.
- The file lives under `examples/`. If a build ever globbed `examples/*.exs` into the test run it would execute these intentionally-failing tests — a CI-hygiene annoyance, not a security issue, and I saw no such wiring.

### Did I verify this?
Yes, by direct re-read of all 196 lines. The conclusion rests on a fact fully determined by this file's own contents — that every value passed to an assertion is an inline literal and there is no external-input channel — so no neighbour module needed to be consulted to establish reachability, and there was no sink to execute and no cross-implementation axis to differentially test. I did not read `ExUnit.Diff`/`ExUnit.Formatter` because no finding or dismissal here depends on their behaviour: the ruled-out diff-renderer entry is settled by the absence of an input path in this file, which reading those modules cannot alter.

### Dropped in re-review
None. Every entry from the first pass is preserved; the prior hedges ("out of scope," "I assumed") are replaced with the grounded reachability basis above, and no dismissal was upgraded to a finding because the file's contents affirmatively rule out a reachable sink (an upgrade would require asserting an input channel the file does not contain).
