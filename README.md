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
