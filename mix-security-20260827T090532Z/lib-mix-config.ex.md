# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/config.ex
<!-- sha256: d32139630e8e879ee63825c0b2a486d9141bc2e1657dd21c245122222e52f291 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/config.ex
<!-- sha256: d32139630e8e879ee63825c0b2a486d9141bc2e1657dd21c245122222e52f291 -->

_Re-analysed on 2026-08-27_

`Mix.Config` is a fully-deprecated delegation shim. Its only in-file logic is `validate!/1` (a non-security completeness gap, reproduced by execution) plus path/wildcard expansion; every dangerous behaviour is a `defdelegate`/delegation to stdlib `Config`/`Config.Reader`, whose config-as-code evaluation is by-design RCE on operator-authored input. In the first pass three dismissals leaned on "outside this file." I re-checked each by (1) reproducing `validate!/1` against the toolchain, (2) tracing every delegation into the installed stdlib copy to confirm the actual sink line, and (3) grepping all of `lib/mix` for callers to confirm no untrusted input reaches this file. Results below.

## Summary

No security finding is reachable through this file. The genuinely dangerous sink (`Code.eval_string` on config-file contents) is reached only by delegation into out-of-scope stdlib modules and only on operator-authored config, which the folder threat model explicitly classes as by-design RCE, not a defended boundary. No dependency/network/archive value crosses into this file: verified by a full-tree caller grep showing zero real callers inside `lib/mix`.

## Security

No findings.

### Ruled out

- **`validate!/1` discards its `Enum.all?/2` result (config.ex:90–112).** Non-`{atom,_}` entries hit the `_ -> false` clause whose `false` is thrown away; the function returns `config` unchanged, so malformed top-level lists pass silently — only a non-keyword *value* under a valid atom key raises. Correctness/dead-defence bug, **not security**: `validate!` is deprecated, has zero callers in `lib/mix`, and its input is developer-authored config (itself executed as code); worst downstream effect is a later crash on malformed developer config, no boundary bypass. Confidence: high.
  - Axes examined: return value on mixed non-tuple list, integer-keyed entry, and valid-key/invalid-value; deprecation status; caller set. | Not examined: none.
  - Executed (Elixir 1.21.0-dev / OTP 29): `validate!([:not_a_tuple, {String, []}])` → `[:not_a_tuple, {String, []}]` (unchanged); `validate!([{123, [a: 1]}])` → `[{123, [a: 1]}]` (unchanged); `validate!([kernel: 123])` → raises `ArgumentError`. Matches the described gap exactly.

- **`Config.__import__!` / `Config.Reader.read!` / `read_imports!` / `__merge__` delegation (lines 41, 51, 60, 65).** These evaluate config files as Elixir source. Reachability and sink confirmed by grepping the installed stdlib copy: `/opt/elixir/lib/elixir/lib/config.ex:365` is `Code.eval_string(contents, [], file: file)`, reached from `Config.__import__!/1` (config.ex:313); `Config.Reader.read!/read_imports!` (reader.ex:100/114) and `Config.__merge__` (config.ex:369) are the real delegation targets. This is arbitrary code execution **by explicit design** (config files are code — the folder threat model lists operator-supplied config as out-of-scope, not a boundary), and the sink lives in stdlib, outside this checkout's scope. Confidence: high.
  - Axes examined: delegation target existence/line; sink identity (`eval_string`); trust class of input (operator config). | Not examined: none — no second implementation exists to diverge against; this is a single delegation chain, verified end-to-end by grep, not an axis comparison.

- **`Path.wildcard`/`Path.expand` in `__import__!/2` and `read_wildcard!/2` (lines 31–38, 74–83).** Pattern originates from `import_config`'s macro argument or a caller-supplied path — the config author, who already holds code execution when the config file evaluates. Wildcard cost, `..` traversal, glob-vs-literal ambiguity are all controlled by that already-trusted author; no untrusted-source path reaches these. Confidence: high.
  - Axes examined: origin of the path argument (macro arg / direct caller); caller set across `lib/mix`. | Not examined: none.

- **`persist/1` → `Application.put_all_env(config, persistent: true)` (lines 68–71).** Global app-env write, but `config` is caller-supplied developer config and persisting it is the documented purpose; no externally-derived key/value crosses a trust boundary. Confidence: high.
  - Axes examined: argument provenance; caller set. | Not examined: none.

## Risks / caveats

- The real code-execution sink is stdlib `Config.__import__!` → `Code.eval_string` at `config.ex:365`, in modules **not in this checkout** (`lib/mix` only). I read the installed `/opt/elixir` copy for reachability confirmation only; I did not security-review those stdlib modules.
- `validate!/1` reproduced directly on the built-from-source toolchain (Elixir 1.21.0-dev / OTP 29). The eval path was not exercised end-to-end because its danger is by-design config-as-code and its sink is out of scope; its reachability was confirmed by tracing the delegation grep, not by feeding it an exploit.
- If a downstream project explicitly aims one of these deprecated APIs at an attacker-controlled file, the result is by-design RCE via config evaluation, not a defect localizable to this file.

## Did I verify this?

- **"No in-repo callers" — verified.** Grepped all of `/Users/lukas/code/claude-scanner/elixir/lib/mix` for `Mix.Config`: only two hits — this file itself, and a documentation string in `mix/tasks/release.ex:640` (telling users to prefer `import Config` over `use Mix.Config`), which is not a call. No task, dep, SCM, or archive path wires untrusted data into these functions. (First-pass hedge "outside this file" resolved: the caller set is empty inside scope.)
- **Delegation sink identity — verified.** `grep` of the installed stdlib confirmed `Config.__import__!/1`, `Config.Reader.read!/2`, `read_imports!/2`, `__merge__/2` all exist and that the terminal sink is `Code.eval_string` (config.ex:365). Reachability confirmed; by-design nature per threat model.
- **`validate!/1` gap — verified by execution**, outputs cited above.

## Dropped in re-review

None. All prior ruled-out entries are retained; the three that carried an "outside this file" hedge are now backed by executed evidence (caller grep + delegation-chain grep + `validate!` reproduction) and their `Not examined` lists are `none`. No dismissal was upgraded to a finding because none involves a two-implementation divergence, and the sole live sink is operator config-as-code, which the folder threat model places out of scope by design.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`Config.__import__!` / `Config.Reader.read!` / `read_imports!` delegation (lines 41, 51, 60).** These evaluate config files as Elixir code (`Code.eval_string`), which is arbitrary code execution — but by explicit design (config files are code), and the sink lives in stdlib `config.ex`/`config/reader.ex`, outside this file's scope. Mix does not feed dependency-controlled config into them.
