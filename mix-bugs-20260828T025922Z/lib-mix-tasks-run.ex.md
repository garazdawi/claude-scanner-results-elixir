# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/run.ex
<!-- sha256: 670e1885a85edbbf1c0ea2d511052fa7725cc5c20c8dca187fbfabc9c2eccfe4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T05:07:42Z_

Based on my review and extensive runtime verification, here is the report.

## Summary

`mix run` (the `Mix.Tasks.Run` task) parses CLI options, rewrites `System.argv`, starts the app, then requires/evals code and optionally runs a script file; I found no reachable behavioural bugs — the argv/option-splitting logic, error paths, `--no-halt`, and the deprecated `--parallel-require` path all behave as documented and as the test suite expects.

## Bugfixes

No findings.

I enumerated candidate defects across every Phase-1 class and drove the observable ones through the actual `mix run` executable built from this source. Everything matched the documented contract and `run_test.exs` expectations:

- **argv/file splitting** (`case {Keyword.has_key?(opts, :eval), head}`): verified `mix run argv.exs foo -- bar` → `["foo","--","bar"]`, `mix run -e ... argv.exs foo -x bar` → `["argv.exs","foo","-x","bar"]`, and `mix run -e ... -- foo` → `["foo"]`, all matching the tests.
- **`--no-halt`**: verified `mix run --no-halt -e ...` keeps the VM alive (killed at timeout, exit 143) while plain `mix run -e ...` exits 0 — the guard `not Keyword.get(opts, :halt, true)` is oriented correctly.
- **deprecated `--parallel-require`**: verified the `IO.warn` fires and the file is still required + eval still runs; `-pr` (parallel+require) also works.
- **error paths**: `-r non-existent` → `Mix.Error "No files matched pattern ..."`; a directory as the file arg → `"No such file: ..."` (via `File.regular?`), matching tests.

## Risks / caveats

- **`@type run_opts` is narrower than the value actually passed to `run/5`.** `run/1` forwards the full `OptionParser` keyword list (containing `:halt`, `:mix_exs`, `:compile`, `:start`, etc.) to `run/5`, whose `@spec` declares the second argument as `run_opts`, which only enumerates `:parallel_require`, `:eval`, `:require`, `:parallel`, `:config`. This is a real spec-vs-reality narrowness that Dialyzer could in principle consume, but `run/5` is `@doc false` (internal), the type is explicitly documented as "illustrative" of the interesting options, `OptionParser` returns a generic `keyword()` anyway, and it has no runtime effect. I did **not** run Dialyzer to confirm it actually warns, so I did not raise it as a finding; noting it here as an unverified, low-value observation.
- **Require/eval interleaving is command-line-order-sensitive**, not "requires always first": I verified `mix run -e 'Helper.hello()' -r mod.exs` fails because the eval runs before the require. Despite the test name "run requires files before evaling commands," this is the intended behaviour — the moduledoc states it "mirrors the command line interface in the `elixir` executable," which processes `-r`/`-e` in order — so it is correct, not a bug. Ruled out.
- I could not run the file's own ExUnit suite in-place (the repo checkout is on a read-only filesystem and the test harness needs to write temp dirs); instead I compiled the tool from this source into a writable build path and drove the real `mix run` binary against a scratch project, which exercises the same code paths the tests assert on.
- `Mix.Project.get()` gating, listener/deps-check/compile flags, and `--no-mix-exs` are forwarded to `app.start` via the raw `args`; that delegation is out of this file's scope and was not audited beyond confirming the forwarding matches upstream structure.

