# Elixir Scan Reports

This repository contains scan reports for Elixir and related components. Each
scan directory has its own `README.md` with a summary of that scan's findings.

## How to read the reports

Start with a scan's README for its scope, evidence basis, and overall
conclusion. Then read its ranked findings:

- Findings are ordered by the scan's stated priority, usually severity and
  confidence. The exact ranking criteria can vary by report.
- Each finding typically identifies an issue, its affected file or files, and
  why it matters. Finding IDs can be used to locate more detail in the
  supporting files in that scan directory.
- Evidence labels such as **verified** indicate that the issue was reproduced
  or otherwise tested; **asserted** indicates that it was assessed without
  direct execution. Check the individual report for the precise meaning.
- Cross-file or cluster findings combine evidence from multiple files. They
  may be described alongside per-file findings, with overlapping items merged.
- Scope limitations and validation notes are scan-specific. Treat the summary
  as applying only to the code and conditions described in that report.

## Note: READMEs are summaries only

The `README.md` in each scan directory is only a summary. It does **not**
contain every issue found; lower-ranked, dismissed, or merged findings may be
omitted. The complete set of findings is in the other `.md` files in each scan
directory (per-file, `CLUSTER_*` and cross-file reports).

To list the finding IDs (`CSCAN-...`) across all reports, together with the file
each appears in (`path:ID`), excluding the summary READMEs, run this from the
repository root:

```sh
grep -rHoE 'CSCAN-[A-Za-z0-9_.-]*[A-Za-z0-9_]' --include='*.md' --exclude='README.md' . | sort -u
```

An ID can appear in several files (e.g. a per-file report and a cluster or
cross-file report), so it may be listed more than once.

Use `grep -rlE 'CSCAN-<id>' --include='*.md' .` to find the files that mention
a given ID. Note that the output may also include IDs that appear only as
examples in report templates or instructions (e.g. `CSCAN-parser-finding-1`).

The pattern requires the ID to end in a letter, digit or `_`, so trailing
punctuation (a sentence-ending `.` or a dangling `-`) is not captured. It was
checked against all `.md` files: every `CSCAN-` heading matches, and the only
non-matching `CSCAN-` mentions are placeholders (`CSCAN-...`,
`CSCAN-<file-basename-without-extension>-<short-mnemonic>`) and one
abbreviated reference (`CSCAN-…-nthtail-overrun`, which is
`CSCAN-elixir_errors-cut-snippet-nthtail-overrun`). To see only the findings
as defined (headings), use:

```sh
grep -rHE '^#+ .*CSCAN-' --include='*.md' --exclude='README.md' .
```

## Example findings

A few of the highest-severity findings, to show what the reports contain.
Each links to the full finding and to the summary it came from.

Upstream status was checked against `elixir-lang/elixir` `main` at
[`168635a97`](https://github.com/elixir-lang/elixir/commit/168635a97) (the scan
was run on tree `3389f8405`). Findings are marked **Fixed upstream** when a
later commit resolves them; the others were still present at that commit.

### Bugs

The only **Critical** finding across all bug summaries is #1; the rest are
verified **High** findings.

1. **Type checker drops bitstring size-expression calls, so valid macros fail
   to compile** (**Critical**, verified) — [`CSCAN-traverse-bitstring-specifier-calls-dropped`](./elixir-bugs-20260827T211546Z/lib-module-types-traverse.ex.md#cscan-traverse-bitstring-specifier-calls-dropped).\
   `of_expr` never traverses the specifier side of a bitstring segment, so a
   local function used in `<<x::size(helper())>>` looks never-called and
   compilation aborts with `undefined function`. It affects every macro
   compilation. Summary: [elixir-bugs](./elixir-bugs-20260827T211546Z/README.md).
2. **`Inspect` for `HashDict`/`HashSet` crashes on every non-empty value**
   (**High**) — [`CSCAN-cluster-inspect-list-two-tuple-return`](./elixir-bugs-20260827T211546Z/CLUSTER_lib-2.md#cscan-cluster-inspect-list-two-tuple-return).\
   A `{doc, opts}` two-tuple is spliced into `concat/1`, which rejects it. It
   is only visible when the three files involved are read together. The
   atom fix in #3 does not address this: `container_doc_with_opts` still
   returns a two-tuple at `168635a97`. Summary:
   [elixir-bugs](./elixir-bugs-20260827T211546Z/README.md).
3. **`HashDict` protocol implementations use the wrong module atom** (**High**,
   verified, **Fixed upstream** in [`13299af2b`](https://github.com/elixir-lang/elixir/commit/13299af2b91a6aff7e430f25849f586c02b3231f),
   2026-10-13) — [`CSCAN-hash_dict-protocol-module-atom-missing-elixir-prefix`](./elixir-bugs-20260827T211546Z/lib-hash_dict.ex.md#cscan-hash_dict-protocol-module-atom-missing-elixir-prefix).\
   The implementations build `:HashDict` instead of `:"Elixir.HashDict"`, so
   every `Enumerable`, `Collectable` and `Inspect` operation raises
   `UndefinedFunctionError`. The same commit fixes `HashSet`. Summary: [elixir-bugs](./elixir-bugs-20260827T211546Z/README.md).
4. **`Registry.lock/3` times out after 5 s instead of waiting** (**High**,
   verified, reproduced) — [`CSCAN-registry-lock-call-timeout`](./elixir-bugs-20260827T211546Z/lib-registry.ex.md#cscan-registry-lock-call-timeout).\
   It uses the default `GenServer.call` timeout rather than `:infinity`, so a
   contender waiting on a long critical section crashes with a `:timeout` exit.
   Summary: [elixir-bugs](./elixir-bugs-20260827T211546Z/README.md).
5. **Stopping `:logger` leaks the Elixir default handler** (**High**,
   verified) — [`CSCAN-app-remove-handler-list-wrapped`](./logger-bugs-20260828T022810Z/lib-logger-app.ex.md#cscan-app-remove-handler-list-wrapped).\
   The removal action is returned wrapped in an extra list and silently
   dropped on stop, so the handler is never removed and the original Erlang
   handler is never restored. Summary: [logger-bugs](./logger-bugs-20260828T022810Z/README.md).

### Security

Only clear security concerns are listed: attacker-controlled input reaching a
code-execution or file-system sink. Denial-of-service and hardening findings
are left out. All of these are **High** and verified.

1. **Lockfile key injection gives code execution from a transitive rebar
   dependency** — [`CSCAN-cluster-lockfile-key-injection-from-dep-name`](./mix-security-20260827T090532Z/CLUSTER_lib-mix-dep.md#cscan-cluster-lockfile-key-injection-from-dep-name).\
   A crafted dependency name is written unescaped into `mix.lock`, which Mix
   later runs through `Code.eval_quoted`. The poisoned lockfile propagates to
   every collaborator. Summary: [mix-security](./mix-security-20260827T090532Z/README.md).
2. **Dependency app-name path traversal deletes and plants files at
   `mix deps.get`** — [`CSCAN-loader-dep-app-name-traversal`](./mix-security-20260827T090532Z/lib-mix-dep-loader.ex.md#cscan-loader-dep-app-name-traversal).\
   A dependency named `'../../X'` makes the checkout path escape `deps/`, and
   `File.rm_rf!` plus `git init` run there before any code is compiled.
   Summary: [mix-security](./mix-security-20260827T090532Z/README.md).
3. **`mix archive.install` zip-slip** — [`CSCAN-archive.install-zipslip-lh-cd-differential`](./mix-security-20260827T090532Z/lib-mix-tasks-archive.install.ex.md#cscan-archiveinstall-zipslip-lh-cd-differential).\
   Path validation checks central-directory names while extraction uses
   local-file-header names, so a crafted `.ez` can write files outside the
   extraction directory. Summary: [mix-security](./mix-security-20260827T090532Z/README.md).
4. **IEx editor command injection** — [`CSCAN-introspection-editor-os-cmd-injection`](./iex-security-20260827T074201Z/lib-iex-introspection.ex.md#cscan-introspection-editor-os-cmd-injection).\
   `open(SomeModule)` builds an editor command with `inspect/1` quoting, which
   is not shell quoting, and runs it with `:os.cmd/1`. A source file named with
   backticks or `$()` executes commands. Summary: [iex-security](./iex-security-20260827T074201Z/README.md).
5. **`Record.extract/2` evaluates untrusted header defaults** — [`CSCAN-extractor-record-default-eval-rce`](./elixir-security-20260827T141433Z/lib-record-extractor.ex.md#cscan-extractor-record-default-eval-rce).\
   Default-value expressions in an Erlang `.hrl` are run through
   `:erl_eval.expr` unrestricted, so a dependency's header can run shell
   commands on the build machine. Summary: [elixir-security](./elixir-security-20260827T141433Z/README.md).

## Reports

| Area | Bug reports | Security reports |
| --- | --- | --- |
| `bin` launchers | [bin-bugs-20260828T073253Z](./bin-bugs-20260828T073253Z/README.md) | [bin-security-20260827T194717Z](./bin-security-20260827T194717Z/README.md) |
| `eex` | [eex-bugs-20260827T205916Z](./eex-bugs-20260827T205916Z/README.md) | [eex-security-20260826T214356Z](./eex-security-20260826T214356Z/README.md) |
| Elixir | [elixir-bugs-20260827T211546Z](./elixir-bugs-20260827T211546Z/README.md) | [elixir-security-20260827T141433Z](./elixir-security-20260827T141433Z/README.md) |
| `ex_unit` | [ex_unit-bugs-20260828T011630Z](./ex_unit-bugs-20260828T011630Z/README.md) | [ex_unit-security-20260827T050345Z](./ex_unit-security-20260827T050345Z/README.md) |
| `iex` | [iex-bugs-20260828T015334Z](./iex-bugs-20260828T015334Z/README.md) | [iex-security-20260827T074201Z](./iex-security-20260827T074201Z/README.md) |
| `logger` | [logger-bugs-20260828T022810Z](./logger-bugs-20260828T022810Z/README.md) | [logger-security-20260827T082043Z](./logger-security-20260827T082043Z/README.md) |
| `mix` | [mix-bugs-20260828T025922Z](./mix-bugs-20260828T025922Z/README.md) | [mix-security-20260827T090532Z](./mix-security-20260827T090532Z/README.md) |
| Workflows | [workflows-bugs-20260828T075203Z](./workflows-bugs-20260828T075203Z/README.md) | [workflows-security-20260827T203009Z](./workflows-security-20260827T203009Z/README.md) |
