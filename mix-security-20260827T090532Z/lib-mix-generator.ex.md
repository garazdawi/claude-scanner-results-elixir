# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/generator.ex
<!-- sha256: beda69a9a756789d490ec6d8d6f88edcb85b1466fac306d7bc04f78ea6618e15 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/generator.ex
<!-- sha256: beda69a9a756789d490ec6d8d6f88edcb85b1466fac306d7bc04f78ea6618e15 -->

_Re-reviewed on 2026-08-27_

I re-read the entire file and resolved the one flagged hedge ("did not exhaustively enumerate every downstream caller") by actually enumerating the callers of the risky entry points via `Grep`, then reading the relevant neighbour module (`mix/tasks/release.ex` and `mix/release.ex`). The two behavioral reproductions from the first pass are preserved. No new findings.

## Summary

`Mix.Generator` is a thin build-time file-generation utility whose sinks (`File.write!`, `EEx.eval_file`, `File.read!`, atom creation) all act on caller-supplied paths/templates that are developer-controlled at build time and cross no dependency/archive/network trust boundary, so no security-relevant weakness is present. The downstream-caller hedge is now discharged: the only Mix-internal callers of the code-executing `copy_template` sink pass operator-project-local template paths.

## Security

No findings.

### Ruled out

- **`copy_template/4` evaluates the template as code (`EEx.eval_file`, line 129).** The embedded Elixir in `source` executes at generation time — verified in the first pass: a template containing `<%= 6*7 %>` / `<%= System.pid() %>` produced `code=42 pid="142"` in the output. This is the documented purpose ("Evaluates and copy templates at `source`"); `source` is a caller-supplied `Path.t()`, and evaluating a trusted bundled template is the intended contract. Passing an attacker-controlled template path is caller misuse, directly analogous to `Code.eval_file` on untrusted input, which the scope excludes. Assigns are only substituted as data (not re-parsed as EEx), so they are not a code-execution vector. **Downstream-caller check (upgraded from hedged assumption in first pass):** `Grep` finds exactly two non-test Mix-internal callers, both in `mix/tasks/release.ex` (lines 1327 and 1424). Both pass `source = Mix.Release.rel_templates_path(release, "...eex")`, which resolves (release.ex:779–781) to `Path.join(release.options[:rel_templates_path] || "rel", path)` — the project's own `rel/` templates directory (operator-controlled, defaulting to the project root), joined with a fixed filename. No dependency-, archive-, git-ref-, or network-derived value reaches the `source` argument; the template file sits at the same trust level as `mix.exs` itself (the operator boundary, out of scope by design). **Confidence: High** that no in-tree caller forwards untrusted input into this sink.
- **No path confinement in `create_file/3`, `create_directory/2`, `copy_file/3` (`File.write!`/`File.mkdir_p!`, lines 40, 48, 75).** Verified in the first pass: `create_file("../escaped.txt", …)` writes outside the cwd. This is by design — `path` is a direct `Path.t()` argument to a file generator that makes no confinement claim. It is not a trust boundary: any caller able to supply the path can equally call `File.write!` itself, so `Mix.Generator` adds no defensible guarantee. Project-name-derived paths (`mix new`) are developer-typed, not attacker-influenced. **Confidence: High.**
- **`Code.format_string!(contents)` when `format_elixir: true` (line 44).** The formatter tokenizes/reformats but does not evaluate code, so no execution occurs; worst case is a `SyntaxError` crash on malformed input. `contents` is caller-generated and the option is only set by callers generating Elixir. **Confidence: High.**
- **`String.to_unsafe_atom("#{name}_template")` / `"#{name}_text"` in the `embed_template`/`embed_text` macros (lines 220, 249).** Atom creation, but it runs at compile time on the macro argument `name`, which lives in the developer's own source (`embed_template(:log, …)`) and is bounded by the number of call sites. Not externally derived; not an atom-table-exhaustion vector. **Confidence: High.**
- **`File.read!` in `copy_file/3` (line 101) and the `[from_file: file]` clauses of both macros (lines 206, 244).** These read a caller/developer-specified file at build time; the path is a documented argument, not attacker-supplied content routed through this file. **Confidence: High.**
- **TOCTOU between `overwrite?/2`'s `File.read` (line 160) and `create_file`'s later `File.write!` (line 48).** The only racer is a co-resident local process, which is outside the Mix attacker model (remote content), and the effect is merely overwriting a file the user was already prompted about. Benign. **Confidence: High.**

## Risks / caveats

- The whole file is a build-time generator; its safety depends on callers passing trusted paths, templates, and contents. That contract is documented, and enumerating the in-tree callers (above) confirms no Mix-internal caller forwards a dependency-, archive-, git-ref-, or network-derived value into the code-executing `copy_template` source or into a `create_file` path. A third-party Mix task outside this source tree that forwarded untrusted input into `copy_template`'s `source` (RCE) or a `create_file` path (traversal) would be a finding in *that* caller, not here.
- `copy_template` assigns are inserted verbatim into the generated file's contents via `String.Chars`; a caller passing attacker-influenced assign values could inject content into the generated artifact (e.g., a config or source file). That is an output-content concern owned by the caller and its provenance, not a code-execution or boundary defect in this file.

## Did I verify this?

- **`copy_template` code execution** — verified by execution in the first pass (Elixir 1.21.0-dev / OTP 29, matching the source tree); `<%= 6*7 %>` → `42`, `<%= System.pid() %>` → the runtime pid. Not re-run this pass; the behavior is stable and the claim unchanged.
- **`create_file` unconfined write** — verified by execution in the first pass (`create_file("../escaped.txt", …)` wrote outside cwd). Not re-run this pass.
- **Downstream-caller enumeration** — verified this pass by reading source, not by execution (a caller inventory is a static fact): `Grep` across `elixir/lib/mix` returned two non-test `copy_template` callers, both in `release.ex`; reading release.ex:1323–1333, 1413–1427 and release.ex:779–781 confirmed both `source` args derive from the operator's `rel/` template directory. This is a caller-graph fact established by reading neighbour modules, which is the correct method for a "which callers exist / what do they pass" question — no runtime axis is involved.

## Dropped in re-review

None. Every prior ruled-out entry is preserved; the single flagged hedge ("did not exhaustively enumerate every downstream caller") was discharged into the verified downstream-caller check on the `copy_template` ruled-out entry rather than dropped, and no ruled-out entry was upgraded to a finding (the enumeration confirmed the benign disposition rather than contradicting it).

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **TOCTOU between `overwrite?/2`'s `File.read` and `create_file`'s later `File.write!` (lines 160, 48).** The only racer is a co-resident local process, which is outside the Mix attacker model (remote content), and the effect is merely overwriting a file the user was already prompted about. Benign.
