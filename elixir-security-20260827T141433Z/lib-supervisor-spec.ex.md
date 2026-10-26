# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/supervisor/spec.ex
<!-- sha256: 9e989b1458a37285010cb2e1afc745523d7b681030261603b47c66de874dc263 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/supervisor/spec.ex
<!-- sha256: 9e989b1458a37285010cb2e1afc745523d7b681030261603b47c66de874dc263 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

Re-review focus: the first pass closed with two "out of scope for this file" hedges about the downstream `Supervisor` that consumes the emitted tuples. Both are now resolved by reading the neighbour module `lib/elixir/lib/supervisor.ex` (the direct and only in-tree consumer of these specs). No new findings resulted; the hedged assumptions upgrade to **verified**. The standing Unicode/compiler-front-end threat model and its RFC checklist do **not** apply to this file — this is a pure Elixir supervision-spec builder with no tokenizer, parser, escape, UTF-8, identifier, or bidi handling of any kind; every applicable sub-claim (BIDI, NFC, mixed-script, escape, grapheme) is not-applicable here.

---

## 1. Summary

`Supervisor.Spec` is a deprecated, purely functional builder that assembles developer-authored worker/supervisor 6-tuples; it reaches no dangerous sink and exposes no untrusted-input path, so it has no meaningful security findings. The re-review confirms via the consumer module that the emitted `{module, function, args}` MFA is forwarded unchanged to Erlang `:supervisor` and applied there, with all four components (`module`, `function`, `args`, `id`) originating from the developer's own supervision-tree code, not from any data the library forwards across a trust boundary.

## 2. Security

No findings.

The module's entire public surface — `supervise/2`, `worker/2,3`, `supervisor/2,3` — only reads a keyword list with `Keyword.get`/`Keyword.put_new`, validates shape, and returns a tuple. There is no code evaluation, no atom creation from strings, no filesystem/network/command/crypto/unicode handling, no shared mutable state, and no concurrency. Error strings interpolate only `inspect/1` output (`get_id/1:195`, `assert_unique_ids/1:201`), which escapes its argument and is bounded by the default inspect `:limit`, so there is no injection or unbounded-formatting sink.

### Ruled out

- **Quadratic `assert_unique_ids/1` (lines 198–206).** `id in rest` is an O(n) membership scan run once per child, giving O(n²) over the child list. **Verified** by measurement against the built Elixir in the first pass: `supervise/2` on unique children took 1.5 ms (n=2 000) → 5.7 ms (4 000) → 21.6 ms (8 000) → 82.8 ms (16 000), i.e. ~4× per doubling — textbook quadratic. Ruled out as a security finding on the **trust boundary**, now confirmed by reading the consumer: the `children` list is populated by developer code — either returned from a `c:Supervisor.init/1` callback (`supervise/2`'s `{:ok, …}` result) or passed positionally to `Supervisor.start_link/2` → `Supervisor.init/2` (`supervisor.ex:785`), whose children flow through `Enum.map(children, &init_child/1)` (`supervisor.ex:813`). No branch of `init_child/1` (`supervisor.ex:816–854`) reads network/disk/dependency data; the list is authored, not forwarded. For an attacker to control the list size they must already control module/function/args (game over), and any such list is *started* afterward — spawning N processes dwarfs the O(n²) dedup (83 ms at 16 000 children vs. the far larger cost of starting 16 000 processes). Latent performance smell in deprecated code, not a DoS primitive.
  - Confidence: high.
  - Axes examined: asymptotic scaling (measured), reachability/trust-boundary (consumer callgraph read). | Not examined: none.
- **Unvalidated `:strategy` / `:max_restarts` / `:max_seconds` values (lines 175–183).** Only presence of `:strategy` is checked; arbitrary atoms/integers pass through into `{{strategy, maxR, maxS}, children}`. **Verified developer-controlled and rejected downstream:** the sibling normalized path `Supervisor.init/2` (`supervisor.ex:786–811`) likewise takes `strategy`/`intensity`/`period` from options without value validation and hands them to Erlang `:supervisor`, which is the authority that rejects invalid flags. No trust boundary crossed; options originate in developer code.
  - Confidence: high.
  - Axes examined: value validation, downstream rejection (consumer read). | Not examined: none.
- **`function` atom used for dispatch (line 284).** `:function` becomes the function applied on the child module. **Verified:** the emitted 6-tuple is passed through `init_child/1` (`supervisor.ex:839–841`) unchanged to `:supervisor`, which applies the MFA; the value is a developer-supplied option (default `:start_link`), no atom is created from a string, and no untrusted value reaches dispatch.
  - Confidence: high.
  - Axes examined: atom-source (literal/developer option), dispatch reachability (consumer read). | Not examined: none.
- **`get_id/1` non-tuple rejection (lines 190–196).** Reachable defensive code (fires on the v1.5 map spec, verified in first pass: `{:map_rejected, …}`), not dead, and raises rather than mishandling — a correctness guard, not a security gap.
  - Confidence: high.
  - Axes examined: reachability, failure mode (raises vs. mishandles). | Not examined: none.

## 3. Upgraded from hedged assumption in first pass

Both first-pass hedges concerned the downstream `Supervisor` consumer and were closed by reading `lib/elixir/lib/supervisor.ex`. Neither produced a finding; both upgrade to **verified**, not to a low-confidence finding, because the neighbour read confirmed the benign disposition rather than leaving it assumed.

- **Hedge 1 — "I did not exercise the downstream `Supervisor` that consumes these tuples."** Verified: the 6-tuple emitted by `child/5` (spec.ex:288) is consumed at `supervisor.ex:839` (`init_child({_,_,_,_,_,_} = tuple) -> tuple`) and, for the `supervise/2` result form, is returned as `{:ok, {flags, children}}` from a `c:Supervisor.init/1` callback straight to Erlang `:supervisor`. The consumer performs **no re-parsing, no atomization, and no transformation** of the tuple — it forwards it verbatim. There is no consumer-side path that turns any tuple component into a new sink.
- **Hedge 2 — "any misuse of the emitted MFA is a property of the consumer."** Verified: the consumer applies no additional validation to the MFA and introduces no new trust boundary; the MFA is applied by Erlang `:supervisor` at child-start time. Every component of the MFA and id originates in developer-authored supervision code. Correctly reformulated (no longer hedged): the emitted MFA is developer-controlled end to end, the consumer forwards it unchanged, and no untrusted-input path exists on either side of the boundary — so there is no security finding to attribute to either module.

## 4. Risks / caveats

- The module is explicitly deprecated; behavior is stable and the first-pass reproduction ran against the sandbox Elixir (a locale warning appeared but did not affect results).
- The quadratic dedup is a genuine, measured behavior; it remains classified as non-security on the confirmed absence of an untrusted-input path in both this module and its consumer. If a downstream project built supervision children directly from attacker-controlled data (an undocumented, self-defeating pattern), the O(n²) cost would still be dominated by the process-start cost, so the residual risk is a minor performance concern, not a DoS primitive.

## 5. Did I verify this?

- **Read the file** in full (spec.ex:1–293). Yes.
- **Read the callgraph neighbour** `lib/elixir/lib/supervisor.ex` (consumer): `init/2` (785–814), `init_child/1` all clauses (816–854), `child_spec/2` tuple-rejection (928–932), `start_child/2` (1038–1055). Yes — this is what closed both hedges.
- **Ran code** for the two hedges? No — the hedges are callgraph/reachability facts about how the consumer treats the tuple, established by reading the consumer, which for a "does the consumer forward vs. transform" question is direct evidence, not a two-implementation differential requiring execution. The one measurement-based claim (O(n²) scaling) was executed in the first pass and its numbers are cited above.
- **Standing Unicode/RFC checklist:** confirmed N/A for this file (no tokenizer/parser/escape/UTF-8/identifier/bidi code present); no sub-claim was force-matched.

### Dropped in re-review

None. Every prior ruled-out entry is preserved; the two hedged caveats were upgraded to verified rather than dropped.
