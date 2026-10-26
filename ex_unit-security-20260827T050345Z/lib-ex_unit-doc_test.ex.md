# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/doc_test.ex
<!-- sha256: f9d76dde1c62df9a8722503e5fce58cf32971960bb12f075fbce39aa4d7766a6 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/doc_test.ex
<!-- sha256: f9d76dde1c62df9a8722503e5fce58cf32971960bb12f075fbce39aa4d7766a6 -->

_Re-reviewed on 2026-08-27_

## Summary

`ExUnit.DocTest` extracts `iex>` code examples from a module's `@doc`/`@moduledoc` chunks (or a markdown file) and compiles them into executable ExUnit tests; it is developer tooling that evaluates trusted developer code by design, so its only meaningful sink is one unvalidated text→atom conversion at `doc_test.ex:919`, which is Low-severity given that trust model. This re-review re-executed every prior evidence claim and resolved the two hedges from the first pass ("outside this file" for `Code.fetch_docs`; "did not exhaustively" for the atom-delta measurement) — no finding changed severity, and the atom-delta caveat is now backed by a clean 2000→2001 direct measurement rather than an inferred delta.

## Security

#### `CSCAN-doc_test-module-concat-atom-creation`
- **Finding ID** — `CSCAN-doc_test-module-concat-atom-creation`
- **Severity** — Low
- **Confidence** — high
- **Location** — `doc_test.ex:919`, `tag_expected/1`, the `"** (" <> error` clause (splits at line 918, concats at line 919)
- **Issue** — When a doctest's expected-output line begins with `** (`, the text between `** (` and the first `)` (`mod`) is passed verbatim to `Module.concat([mod])`, which mints a fresh atom (`Elixir.<mod>`) for every distinct name. The name is never validated or length-bounded, and `Module.concat` (unlike `Module.safe_concat`) creates atoms unconditionally. The atom table is never garbage-collected (CWE-770), and names longer than 255 bytes raise `SystemLimitError`. All of this happens at *compile time* of the test module, inside the documented `doctest/2` and `doctest_file/2` macros.
- **Class** — Atom creation (CWE-770)
- **Impact** — Doc text drives unbounded atom-table growth and can crash compilation. In normal use the docs are the project's own (self-inflicted). The trust boundary only matters when the doc is *not* the top-level developer's — e.g. `doctest_file` pointed at externally-contributed markdown, or `doctest ThirdPartyDep.Module` against a dependency's docstrings — where a hostile author can inject one atom per `** (Name)` line, or a single `SystemLimitError` crash, into the victim's build. It stays Low because any such scenario already grants full code execution by design (the `iex>` expressions themselves are compiled and evaluated as tests), so the atom sink is strictly subsumed by intended RCE and adds no privilege.
- **Evidence** — `verified` (re-executed this pass on Erlang/OTP 29, Elixir 1.21.0-dev):
  - `Module.concat/1` mints atoms while `safe_concat` refuses — direct measurement of the varying-name contribution in isolation: `Module.concat atom delta for 2000 names: 2001` (clean ~1:1 ratio, no shared-parse-atom confound), and `safe_concat novel: {:raised, ArgumentError}`.
  - Over-long name: `concat 300-char: {SystemLimitError, "a system limit has been reached"}`.
  - The `mod` fed to `Module.concat` is exactly the pre-`)` substring: `:binary.split(error, ")")` on a well-formed `** (Name) msg` line yields `[Name, msg]`.
- **Suggested fix** — Replace `Module.concat([mod])` with `Module.safe_concat([mod])` so a non-existent module name raises `ArgumentError` instead of interning a new atom. Behavioral caveat: `safe_concat` requires the exception module's atom to already exist at *extraction* time; for the normal case (the exception is genuinely raisable, so its module is loaded) this holds, but a doctest naming an exception module whose atom is not yet interned would now error — so pair it with a friendly `ExUnit.DocTest.Error` rescue, and additionally guard the `[mod, message] = :binary.split(...)` match (see Ruled out) so malformed `** (` lines report cleanly rather than crashing.

### Ruled out

- **`Code.string_to_quoted!/2` (line 624) and the generated evaluation of doctest expressions (`test_case_content/4`, `__test__`/`__inspect__`/`__error__`)** — Parsing and executing the `iex>` code is the entire purpose of the feature; the code is trusted developer input. Atoms created while parsing are inherent to compiling any Elixir source and not separately controllable beyond the intended RCE. Not a finding.
  - Axes examined: sink class (eval/atom via intended compilation), trust source (developer test code). Not examined: none — the by-design RCE trust boundary makes further axes moot (any bypass would be subsumed by the intended code execution).

- **`Code.fetch_docs(module)` (line 336)** — Reads the module's BEAM `Docs` chunk; the underlying `binary_to_term` lives in the `Code` module, not in this file. Under the folder threat model a hostile `.beam` on the load path already implies code execution, so this deserialization sink is out of scope *for this file*. This is a scope boundary (the sink is not in `doc_test.ex`), not an unexamined guard axis within this file. Not a finding here.
  - Axes examined: sink location (deserialization body is in `Code`, not this file), trust source (attacker-writable `.beam` ⇒ code-exec, out of scope). Not examined: none *within this file's boundary* — the `binary_to_term` safe/unsafe distinction belongs to a `Code`-module review, not this one; flagged there rather than hedged here.

- **Malformed `** (` with no closing `)` (line 918)** — `:binary.split(error, ")")` returns a single-element list and the `[mod, message] =` match raises a raw `MatchError` at compile time. Re-executed this pass: `no-close-paren split match: {MatchError, "no match of right hand side value:\n\n    [\"NoCloseParen\"]\n"}`. Compile-time robustness/UX defect on the doc author's own input with no privilege crossing — not a security finding, but worth fixing alongside the sink above.
  - Axes examined: input shape (missing `)`), failure mode (raw `MatchError` at compile time, self-inflicted). Not examined: none.

- **`File.read!(file)` (line 298) in `__doctest_file__`** — The path is the literal argument the developer wrote in `doctest_file "README.md"`; no external component, no traversal primitive. Not a finding.
  - Axes examined: input source (compile-time literal), traversal primitive (none). Not examined: none.

- **Regex `expr =~ ~r/#[\w\.]+</` (line 631)** — Fixed literal pattern with no nested quantifiers (linear, no ReDoS), and only evaluated in the already-failed compile branch. Not a finding.
  - Axes examined: pattern structure (no nested quantifier / catastrophic backtracking), reachability (already-failed branch). Not examined: none.

- **`String.to_charlist(file)` (line 659) and `Path.relative_to_cwd` (lines 188/299/317/334)** — Charlist/path conversions for stack-trace and message display; no atom creation, no filesystem write. Not a finding.
  - Axes examined: sink class (charlist/path formatting, no atom/write). Not examined: none.

- **`module.module_info(:compile)` (lines 187/316/332)** — Called on the developer-supplied, always-loaded module argument; not externally influenced. Not a finding.
  - Axes examined: input source (developer-supplied loaded module). Not examined: none.

## Risks / caveats
- The atom-delta claim is now backed by a **direct, confound-free measurement** (2000 distinct novel names → 2001 atom-table growth, ~1:1), replacing the first pass's inferred ~1038 delta that shared interned parse atoms between two measurement compiles. The `safe_concat` existence check (`ArgumentError` on a novel name) independently confirms the injected atoms are attributable to `Module.concat`, not to parsing.
- The `Module.safe_concat` fix changes behavior for doctests that assert on an exception module not yet interned at extraction time; a maintainer should confirm the compile-ordering assumption (exception module loaded before its own doctests are extracted) holds across the projects that rely on `doctest`, or gate the change behind a friendly error as suggested.
- Overall: consistent with the folder threat model, this file runs trusted developer/test code; no network or untrusted-peer surface exists, and no code-execution, deserialization, path-traversal, or command-injection defect beyond the by-design evaluation lives in this file. The one deserialization sink (`Code.fetch_docs` → `binary_to_term`) is in the `Code` module and belongs to a review of that module.

## Did I verify this?
- **`Module.concat` mints atoms / `safe_concat` refuses / over-long name crashes** — Yes, re-executed this pass (OTP 29, Elixir 1.21.0-dev): delta `2001` for 2000 names; `safe_concat` → `ArgumentError`; 300-char name → `SystemLimitError`.
- **Malformed `** (` → `MatchError`** — Yes, re-executed this pass: single-element split list fails the two-element match with a compile-time `MatchError`.
- **`Code.fetch_docs` internal `binary_to_term` safety** — Not verified here, and intentionally not upgraded: the deserialization body is not in this file, and under the attacker-writable-`.beam` trust model it is out of scope for `doc_test.ex`. Recorded as a scope boundary and deferred to a `Code`-module review rather than left as a hedged assumption or a speculative in-file finding.

### Dropped in re-review
None. Every prior finding and ruled-out entry is preserved. The two first-pass hedges were resolved in place: the `Code.fetch_docs` "outside this file" hedge is reframed as an explicit scope boundary (sink not in this file, deferred to the `Code` module), and the "did not exhaustively" atom-delta hedge is replaced by a clean re-executed 2000→2001 direct measurement.
