# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.get.ex
<!-- sha256: c96ea90c9782a3539d8747de125e4479278a6115ef03f0b0d0fe01f50452395d -->

_Analysed on 2026-08-27T10:47:52Z_

All internal callers confirm the point: the only two internal invocations pass either **no args** (`Mix.Task.rerun("deps.get")` in mix.ex) or `["--only", Atom.to_string(Mix.env())]` (installer.ex) — where `Mix.env()` is an already-interned, operator-configured atom. The only way an arbitrary string reaches the `--only`/`--target` value is the operator typing it on the command line. That matches the threat model's explicitly out-of-scope "operator themselves," and mirrors upstream Elixir's identical `:"#{value}"`.

I have fully traced the single sink in this file. Here is the report.

## Summary

`Mix.Tasks.Deps.Get` is a thin CLI wrapper around `Mix.Dep.Fetcher.all/3` whose only security-relevant sink — dynamic atom creation from an option value — is reachable only with operator-supplied command-line input, so there are no findings.

## Security

No findings

### Ruled out

- **Dynamic atom creation via `String.to_unsafe_atom("#{value}")` (`deps.get.ex:41`, `do_run/1`).** This is a genuine atom-table sink (`String.to_unsafe_atom/1` = `:erlang.binary_to_atom(string, :utf8)`, no GC — CWE-770), and a value over 255 code points raises `SystemLimitError` (verified: `String.to_unsafe_atom(String.duplicate("a",300))` → `SystemLimitError`). But `value = opts[switch]` for `switch ∈ {:only, :target, :check_locked}` traces back through `OptionParser.parse` to `run/1`'s `args`, i.e. the `--only`/`--target` flags typed on the command line. Both internal callers of the task pass non-hostile input: `Mix.Task.rerun("deps.get")` (no args → empty `fetch_opts`, verified) and `Mix.Task.run("deps.get", ["--only", Atom.to_string(Mix.env())])` (an already-interned environment atom). No dependency, lockfile, archive, git ref, or network peer can steer this value. Reaching atom exhaustion or the crash requires the operator to feed their own build ~1M distinct values or one oversized value — self-inflicted, and the operator already has full RCE via `mix.exs`. This is the threat model's out-of-scope "operator themselves," and is byte-for-byte the behavior of upstream Elixir's `:"#{value}"`. Not a vulnerability.
- **`Mix.Task.run("archive.check", args)` gated on `"--no-archives-check" not in args` (`deps.get.ex:23-25`).** `--no-archives-check` is a documented flag; the forwarded `args` are the same operator CLI args, and no OS process or path sink lives in this file. Skipping the archive check is documented behavior, not a bypass.
- **`Mix.shell().info("All dependencies have been fetched")` (`deps.get.ex:46`).** Constant string; no untrusted value interpolated into the log/shell line.
- **Comprehension nil-handling.** Verified `value = opts[switch]` acts as a truthiness filter, so absent options are skipped (result `[]`) rather than producing a spurious `:""` atom — no correctness/security gap.

## Risks / caveats

- **Out of this file's scope (not reviewed here):** the actual fetch, checksum/redirect/TLS handling, and any zip-slip/traversal live in `Mix.Dep.Fetcher.all/3`; lockfile parsing/deserialization lives in `Mix.Dep.Lock.read/0`; archive trust lives in `archive.check`. This file merely forwards to them. Those files need their own review for the network/archive/deserialization sinks the folder threat model flags.
- **Undocumented switch:** `--target` is parsed (`switches: [target: :string]`) but not listed in the `@moduledoc`. This is a documentation gap, not a security issue — its value flows only into the same operator-controlled atom sink already analyzed.
- **Reproduction:** ran against the toolchain built from the source under review (`/opt/elixir`). Confirmed atom-count delta of 1 per distinct value, `SystemLimitError` on a 300-char value, and correct nil-filtering; I did not exercise the full `mix deps.get` end-to-end against a live project, which is unnecessary since the only in-file sink and its trust boundary were settled directly.

