# Security Scan Summary — ExUnit (`elixir/lib/ex_unit`)

**Scope & method.** 20 source files reviewed (ExUnit test framework: runner, formatters, capture helpers, callbacks, filters, manifest, doctest, examples). No `VALIDATION.md` was produced. `CROSS_FILE.md` records **no cross-file finding** and all three `CLUSTER_*.md` record **no cluster finding** — every concrete cross-file dependency was resolved by the per-file reviewer reading the cited neighbour, and the two load-bearing sinks are correctly localized in the files that own them. Five findings survive across the per-file reports (**1 Medium, 4 Low**, all high-confidence).

**Governing trust model.** Every reviewer applied ExUnit's premise: it executes *trusted developer test code*, so the test author already has arbitrary code execution. Consequently most sink-shaped code is "self-inflicted" and out of scope. The findings that matter are the ones with an **external-data channel that crosses into the trusted run** — chiefly the on-disk failures manifest.

---

## 1. Unsafe `binary_to_term` on the failures manifest — **Medium** (high confidence)
`CSCAN-failures_manifest-unsafe-binary-to-term` · `failures_manifest.ex:71` (`safe_binary_to_term/1`, reached from `read/1`:47 and `info/1`:57)

The `mix test --failed` manifest is decoded with `:erlang.binary_to_term/1` **without `[:safe]`**. Despite the name, the wrapper only `rescue`s `ArgumentError` — it does nothing to stop the decode from **minting arbitrary atoms, funs, pids and refs**. Critically, the decode runs to completion *before* the `{@manifest_vsn, %{} = manifest}` shape check, so the version/shape "validation" is not a guard (`:safe` is). Reached on **every `mix test`** via `ExUnit.RunnerStats` on `:suite_started` whenever `:failures_manifest_path` is configured (Mix's default).

- **Impact:** an attacker who can write the manifest (poisoned CI cache that restores `_build`, a repo/tarball shipping a crafted `_build`, or a co-located writer) makes the next `mix test` decode hostile bytes. A ~1 MB manifest of ~1M unique atoms exhausts the atom table (default 1,048,576) and crashes the BEAM running the suite — DoS on dev machines and CI runners; sub-limit files leak atoms permanently (CWE-502 → CWE-770). On exhaustion `binary_to_term` raises `:system_limit`, which the narrow `rescue ArgumentError` does **not** catch, so it escapes as an uncaught crash.
- **Verified:** planted manifest injected 50,031 novel atoms in one `read/1`; a bad-version file still injected its atoms (decode-before-validate confirmed); a decoded fun was materialised and invocable (though consumers never call it, so DoS, not RCE).
- **Fix:** decode with `[:safe]` (legitimate manifests contain only already-loaded atoms, so nothing valid is rejected); optionally broaden the rescue to `SystemLimitError`.
- **Corroboration:** the cross-file ledger confirms both reach paths independently — `runner_stats.ex` (`:suite_started` → `read`) and `filters.ex` (`failure_info/1`/`fail_all!/1` → `info`) both point here as the real sink. This is the scan's most impactful and best-substantiated finding.

---

## 2. Exponential-time list-diff blow-up — **Low** (high confidence)
`CSCAN-diff-nested-list-exponential-rediff` · `diff.ex:644` (`list_script_to_diff/6`) + `diff.ex:606` (`follow_snake/1`)

When two lists are diffed, each *unequal* element pair is diffed twice and the first result is discarded rather than memoised; for nested lists this compounds to ≈2^depth cost. Specific to the list path (tuples/maps diff each element once).

- **Impact:** a failing `assert a == b` whose sides are lists nested ~23+ levels deep and differing anywhere burns a CPU core exponentially. In normal use the sole caller (`formatter.ex:561`) wraps it in `Task.yield(1500) || Task.shutdown(:brutal_kill)`, bounding it to ~1.5 s + a dropped diff per failing assertion. Where expected values derive from untrusted data (golden-file/JSON-array comparisons, property tests), an attacker who can force such a failure imposes the 1.5 s burn + degraded output repeatedly; **called directly, `Diff.compute/3` (which is `@doc`'d) has no timeout and hangs unboundedly.**
- **Verified:** measured ~1.95×/level growth; depth-24 mismatched lists → 2,972 ms; end-to-end via the formatter's `find_diff/3` → 1502 ms and `nil` diff.
- **Fix:** memoise element-pair diffs per `compute` run (or thread `follow_snake`'s result through the edit script); optionally add a depth/step bound in `myers_difference_list`.

---

## 3. Linked diff `Task` crash kills the shared formatter — **Low** (high confidence)
`CSCAN-formatter-find-diff-linked-task-crash` · `formatter.ex:558-565` (`find_diff/2`)

The diff runs in a **linked** `Task.async(Diff, :compute, …)`. If `Diff.compute` exits abnormally, the link propagates the exit and kills the caller **uncatchably** (a surrounding `try/rescue/catch` cannot intercept a link exit); if the caller traps exits, `Task.yield` returns `{:exit, reason}`, which matches neither `case` branch → `CaseClauseError`. The 1500 ms timeout contains *slowness*, not *crashes*.

- **Impact:** formatting runs inside the `ExUnit.CLIFormatter` GenServer's `handle_cast`, and that GenServer does **not** trap exits (verified — `init/1` returns `{:ok, config}` with no `Process.flag(:trap_exit, true)`). A single diff-task crash therefore kills the shared formatter, disrupting reporting for the rest of the run — the "one bad test breaks its siblings" outcome the isolation model exists to prevent.
- **Reachability keeps it Low:** executed fuzzing (1,682 `==`/`===` pairs + 2,400 match combos) found **no** crash driven by the untrusted `right` value alone; every reproduced crash (missing-pin `MatchError` via `diff_pin/3`; unknown-context `FunctionClauseError`) is carried by the **developer-supplied `context`/`pins`**, which the real `assert` macros always emit consistently. A future input that crashes `Diff.compute` from `right` alone under a well-formed context would upgrade this to Medium.
- **Fix:** use `Task.Supervisor.async_nolink/3` (or `spawn_monitor`) so no link exists, and add `{:exit, _} -> nil` to the `case` — turning a diff crash into a missing diff rather than a dead formatter.

*(Findings 2 and 3 are the two endpoints of the single `diff.ex`↔`formatter.ex` sink the cross-file ledger flagged as load-bearing; the 1.5 s task wrapper is the shared mitigation for both, and any refactor that calls `Diff.compute/3` directly removes it.)*

---

## 4. Unbounded atom creation from doctest exception names — **Low** (high confidence)
`CSCAN-doc_test-module-concat-atom-creation` · `doc_test.ex:919` (`tag_expected/1`)

A doctest expected-output line starting with `** (` has the text up to the first `)` passed verbatim to `Module.concat([mod])`, which mints a fresh atom per distinct name (unlike `Module.safe_concat`), unbounded and un-length-checked; names >255 bytes raise `SystemLimitError` at compile time (CWE-770). All at *compile time* of the test module.

- **Impact:** self-inflicted for a project's own docs, but the trust boundary bites when the doc is **not** the top-level developer's — `doctest_file` pointed at externally-contributed markdown, or `doctest ThirdPartyDep.Module` against a dependency's docstrings — where a hostile author injects one atom per `** (Name)` line, or a single `SystemLimitError` build crash. Stays Low because any such scenario already grants code execution by design (the `iex>` expressions are compiled and run as tests), so the atom sink adds no privilege.
- **Verified:** 2,000 distinct names → 2,001 atom-table growth (~1:1); `safe_concat` refuses novel names; 300-char name → `SystemLimitError`. A malformed `** (` with no closing `)` also raises a raw compile-time `MatchError`.
- **Fix:** use `Module.safe_concat/1` (paired with a friendly error), and guard the `[mod, message] = :binary.split(…)` match.
- **Coverage gap (from ledger, Unresolved):** the related `Code.fetch_docs` → `binary_to_term` sink lives in the `Code` module, outside this checkout — deferred to a `Code`-module review, not adjudicated here.

---

## 5. Atom creation from filter keys — **Low** (high confidence)
`CSCAN-filters-cli-key-atom-creation` · `filters.ex:137-138` (`parse/1`)

`parse/1` converts every filter *key* from a raw string via `String.to_unsafe_atom/1` (= `binary_to_atom/2`), an unchecked non-existing-atom creation; each distinct key mints a permanent, never-GC'd atom (CWE-770).

- **Impact / reachability:** self-inflicted under the documented model — the sink is executed-verified (`parse(["foo"]) → [:foo]` interns a fresh atom), but its *untrusted reachability is unconfirmed*: the documented callers (`parse/1`, `parse_paths/1`, `failure_info/1`, `fail_all!/1`) live in `mix/tasks/test.ex`, which is **absent from this checkout** — the reviewer could not confirm only developer-CLI strings reach the sink. `parse/1` is documented-public, so any wrapper feeding it externally-sourced strings inherits the sink.
- **Fix:** prefer `String.to_existing_atom/1` for the bare-key branch with a graceful skip, or route keys through an allow-list.
- **Coverage gap (from ledger, Unresolved):** the `mix/tasks/test.ex` caller could not be verified — carried as this finding's standing caveat.

---

## Files with no findings
No security defects in: `assertions.ex`, `callbacks.ex`, `capture_io.ex`, `capture_log.ex`, `capture_server.ex`, `case.ex`, `case_template.ex`, `cli_formatter.ex`, `event_manager.ex`, `on_exit_handler.ex`, `on_exit_handler/supervisor.ex`, `runner.ex`, `runner_stats.ex`, `server.ex`, `ex_unit.ex`, `mix.exs`, and the two `examples/*.exs` scripts. In each, the only sink-shaped code (dynamic dispatch, `to_unsafe_atom` on bounded internal counters, `:public` ETS, `Path.expand`+`rm_rf!`, module dispatch) is driven exclusively by trusted developer/compile-time input, with provenance now verified by reading neighbours rather than assumed.

## Notable latent / robustness items (out of security scope, flagged by reviewers)
- **`runner.ex` `create_tmp_dir!`** — `Path.expand` + `File.rm_rf!` on the `:tmp_dir` tag has **no root-confinement**; a `../../..` `extra_path` was proven to escape (`⇒ /tmp/pwned`). Safe *today only* because the tag is developer-controlled (verified against `case.ex`); it would become an arbitrary-delete/zip-slip primitive if any future caller routes untrusted input into that tag. A defensive `Path.safe_relative` would harden it.
- **`capture_io.ex`** — dead-pid `StringIO` process leak; **`event_manager.ex`** — `stop/1` hangs suite shutdown if a formatter blocks in `terminate/2`; **`cli_formatter.ex`** — `ArithmeticError` div-by-zero under `--slowest` with a 0 µs run; **`assertions.ex`** — `assert_raise/3` calls `Exception.message/1` unguarded. All cross no trust boundary; noted for a general-correctness pass.
