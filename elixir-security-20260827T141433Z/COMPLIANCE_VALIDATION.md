# Validator verdicts

Automated prove-or-disprove pass over every `CSCAN-*` finding in the
sibling per-file reports. Each verdict is one Claude call with both-
sides steelman prompt at `--effort medium`. Read the **Reasoning**
line per finding to decide whether to trust the verdict.

## Summary

| Verdict | Count |
|---|---|
| real | 1 |
| fp | 0 |
| uncertain | 0 |
| unknown (validator failed) | 0 |
| **total** | **1** |

## Real findings (1)

<!-- cscan-verdict CSCAN-elixir-rfc-directional_formatting_not_prohibited real -->

### Validator verdict for CSCAN-elixir-rfc-directional_formatting_not_prohibited

**Verdict:** real
**Confidence:** medium
**Files read:** /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_interpolation.erl, /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_tokenizer.hrl, /Users/lukas/code/claude-scanner/elixir/lib/elixir/src/elixir_tokenizer.erl

**Steelman (real):** The code matches the finding exactly. `?bidi/1` (tokenizer.hrl:27-35) enumerates precisely the nine directional formatting characters (202A/B/C/D/E, 2066-2069). In `extract_char` (interpolation.erl:95-102), the first raw bidi char in any string literal yields an unconditional `{error, ...}` ("invalid bidirectional formatting character in string…"), and `tokenize_comment` (tokenizer.erl:1357-1358) does the same for comments. These are core, always-on tokenizer paths — every Elixir source string and comment flows through them in production, no flag or test hook gating them. There is no exception for balanced/terminated isolates or embeddings, and no warn-based alternative. UTS #55 §5.1.6 SHOULD-NOT says "Implementations should not prohibit the use of the directional formatting characters," recommending warnings scoped to when their effect spans a token/comment rather than outright rejection. Elixir outright rejects every raw occurrence, which is a genuine, reachable deviation from the cited clause.

**Steelman (FP):** The `\uXXXX` escape hatch means the codepoints can still reach runtime string data, so one could argue "use" is not fully prohibited — satisfying §5.1.6 in spirit. More strongly, §5.1.6 is a SHOULD-NOT that explicitly coexists with the Trojan-Source/spoofing concern; Elixir's rejection is a deliberate, standards-aware tradeoff prioritizing UAX #9 spoofing defense, and UTS #55 tolerates stricter profiles that reject unbalanced formatting. If one reads the clause as governing only wholesale language-level banning (e.g. from identifiers) rather than requiring raw acceptance in string/comment literals, the obligation arguably doesn't bind this path (FP category c).

**Reasoning:** The escape hatch does not rescue the clause: comments have no escaping mechanism at all (tokenizer.erl:1357 hard-errors with no alternative), and even in strings the escaped form does not render as RTL in the *source*, which is exactly the display use §5.1.6 wants preserved — so raw use is genuinely prohibited on a live production path. The rubric is explicit that a real, reachable deviation from a cited normative clause remains `real` even when it is a defensible, deliberate tradeoff and even when its only consequence is loss of a legitimate benign capability; "safer/more restrictive" is not grounds for FP. The recommended mitigation (scoped warning) is verifiably absent. The only real hesitation is interpretive — whether §5.1.6's "should not prohibit" binds this specific literal path — which is why confidence is medium rather than high, but the plain reading of both the clause and the code supports a `real` verdict at Low severity.

