# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/local.hex.ex
<!-- sha256: d4af8ab22774b0b267041553c67820d5e835f637e7da9f84bc09abcd4528e414 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/local.hex.ex
<!-- sha256: d4af8ab22774b0b267041553c67820d5e835f637e7da9f84bc09abcd4528e414 -->

_Re-analysed on 2026-08-27_

## Summary

`Mix.Tasks.Local.Hex` is a thin orchestration wrapper. Its only externally-influenced inputs — the mirror CSV's version fields (`elixir_version`, `hex_version`, `otp_release`) and SHA-512 `digest` — are neutralized by a host-fixed URL template and a checksum gate that this file wires up and that the downstream code (`installer.ex` → `utils.ex`) honours. No security vulnerability is reachable through the documented public API (`mix local.hex [version]`). **No findings.**

In this pass every hedge from the first-pass report ("replica of checksum", "belongs to those files' reviews", "only matches ≥4 columns", "`Mix.Hex.url` permits http") was resolved by reading the actual neighbour modules and by executing each axis against the real functions. Results below cite what was read and what was run.

## Security

No findings.

### Ruled out

- **URL/SSRF injection via CSV version fields** (`run_install`, lines 78–83).
  The peer-controlled `elixir_version`/`hex_version`/`otp_release` are `String.replace`d into `hex_url <> @hex_archive_path`, i.e. strictly into the URL *path*, after the `https://builds.hex.pm` authority is already fixed by `Mix.Hex.url/0`.
  Confidence: High.
  Axes examined: host-substitution, scheme-substitution, path-escape (`../`), userinfo/`@`-smuggling, embedded-scheme, CRLF. | Not examined: none.
  Evidence (RAN, OTP29): with `elixir_version ∈ {"x@evil.com", "../../../..", "foo/@evil.com", "://evil.com/x", "\r\nHost: evil"}` the constructed URL always yields `URI.parse` `host="builds.hex.pm"`, `scheme="https"`. Authority cannot be moved.

- **Checksum bypass / "no digest → accept anything" branch** (forwarding at line 88).
  This file *always* emits `"--sha512", sha512`, and `sha512` is always a binary (never `nil`).
  Confidence: High.
  Axes examined: nil-digest reachability, empty-digest (`""`), wrong-digest, matching-digest. | Not examined: none.
  Evidence:
  - **Read** `local.ex:219–228`: `find_by_elixir_version/3` matches only `[artifact_version, digest, hex_elixir_version, hex_otp_release | _]` (≥4 columns), and `find_matching_versions!` returns `{hex_elixir_version, artifact_version, digest, hex_otp_release}` → `sha512 = digest`, a binary. The `nil` slot cannot be produced from this file.
  - **Read** `utils.ex:832–848`, the *real* `checksum/1` (not a replica): the accept-anything path requires `opts[:sha512] == nil`.
  - **Read** `installer.ex:71–134` + `archive.install.ex:78–91`: `--sha512` (`sha512: :string`) is parsed into `opts`; a `.ez` URL takes the `{:url, src}` branch → `local_install/3` → `Mix.Utils.read_path(src, opts)`; `module.install` (the `:zip.extract` sink) runs only on `{:ok, binary}`. The `:fetcher` branch that raises "--sha512 not supported" is not reached for a URL spec.
  - **RAN** the verbatim `checksum/1` body against a real SHA-512: `sha512: nil → {:ok, bin}`; `sha512: "" → {:checksum, :mismatch}`; `sha512: "deadbeef" → {:checksum, :mismatch}`; `sha512: <correct> → {:ok, bin}`. An empty or wrong digest fails closed *before* extraction; the bypass branch is unreachable from this file.

- **Option injection via a hostile digest** (line 88).
  A mirror-supplied `sha512` cannot inject an extra switch into `Archive.Install`'s strict `OptionParser`.
  Confidence: High.
  Axes examined: embedded-space splitting, leading `--`, leading `-`. | Not examined: none.
  Evidence (RAN, strict switches `[force: :boolean, sha512: :string]`): `"aa --submodules"` parses as one literal string value; `"--submodules"` and `"-x"` raise `OptionParser.ParseError` (fail-closed crash, not injection). No path installs unverified code, and a malicious mirror can already trivially fail the command.

- **`Version.compare(Hex.version(), version)` crash** (line 54).
  `version = List.first(args)` is operator-supplied argv; an invalid version raises `Version.InvalidVersionError` — operator self-harm on their own command line.
  Confidence: High.
  Axes examined: internal-caller reachability, external-input reachability. | Not examined: none.
  Evidence: **Read** `hex.ex:20,48` — the internal callers `ensure_installed?`/`ensure_updated?` invoke `Mix.Tasks.Local.Hex.run(["--force"])` with no version arg, so `version` is `nil` and line 54 is not entered from them. Only a human-typed CLI argument reaches it. Out of scope (operator boundary).

- **`:code.purge(Hex)`** (line 86) — no external input; local module lifecycle management. Confidence: High. Axes examined: input source. | Not examined: none.

- **Checksum-before-extract / zip-slip guard / TLS verification** — invoked correctly here; the mechanisms live downstream and were confirmed present: `read_path` gates extraction on `{:ok, binary}` (`utils.ex:808–823`), `installer.ex:131–134` only calls `module.install` on `{:ok, binary}`, and `Archive.Install.validate_archive_path!` / `read_httpc` (`verify: :verify_peer`) are their own files' responsibility. This file wires them correctly. Confidence: High.

## Risks / caveats

- **Plaintext mirror is possible but outside this file's boundary.** **Read** `hex.ex:79–81`: `Mix.Hex.url/0` returns `System.get_env("HEX_BUILDS_URL") || System.get_env("HEX_MIRROR") || "https://builds.hex.pm"` verbatim, permitting `http://`. Over plaintext the digest offers no MITM protection (attacker controls both CSV and archive on one channel). This is operator/environment configuration, not dependency/archive/peer data, and `local.hex.ex` contains no code that weakens the scheme — flag for the `Mix.Hex` review, not a finding here. Confidence: High (verified by read).

- **CSV-derived path bytes reach `httpc` unescaped.** `hex_version` (= `artifact_version`, the 2nd-precedence CSV field) is only constrained by `hd(&1) == artifact_version` when a version arg is given, and is otherwise free bytes spliced into the URL *path* (lines 79–82). It cannot alter host/scheme (shown above), but CRLF/control bytes in the path are handed to `Mix.Utils.read_httpc` → OTP `:httpc`. Request-line sanitization is that module's responsibility; OTP `httpc` validates/encodes the request target. Noted as a cross-file consideration, not a finding in this file. Confidence: Medium (sink not in this file; not driven end-to-end).

- **Network flow not exercised end-to-end.** The sandbox has no route to `builds.hex.pm`, so URL construction, the checksum gate, and option parsing were reproduced against the *real* Elixir/OTP29 code locally rather than by driving a live `mix local.hex`. The behavioral claims above are executed and verified; the live fetch was not.

- **Malformed-CSV robustness is a `local.ex` concern.** `find_by_elixir_version/3` (`local.ex:219`) has no catch-all clause, so a CSV row with <4 columns raises `FunctionClauseError` during parsing. That crash sink is in `mix/local.ex`, not this file; `local.hex.ex` only consumes the already-formed 4-tuple. Flag for the `local.ex` review. Confidence: High (verified by read).

## Did I verify this?

- **Read** the target file in full and five neighbour modules named in the callgraph: `mix/hex.ex` (`url/0`, internal callers), `mix/local.ex` (`find_matching_versions!`, `find_by_elixir_version`, `parse_csv`), `mix/utils.ex` (`read_path`, real `checksum/1`, `hexhash`), `mix/local/installer.ex` (`install/3`, `local_install/3`, the `--sha512` branch), `mix/tasks/archive.install.ex` (switches).
- **Ran** (Elixir/OTP29 in-sandbox): URL-substitution across 5 hostile version values (host/scheme unchanged); the verbatim `checksum/1` body across nil/empty/wrong/correct digests (fails closed except on correct); strict `OptionParser` across 3 hostile digest values (no switch injection; fail-closed on `--`/`-`).
- Every axis on every ruled-out entry was actually tried; none rely on an unread invariant. `Not examined: none` on all four security ruled-outs.

## Dropped in re-review

None. The first pass reported no findings; all ruled-out entries are preserved, and each first-pass hedge ("replica", "only matches ≥4 columns", "belongs to those files' reviews", "`Mix.Hex.url` permits http") is now backed by a read of the real neighbour module and/or an executed axis rather than an assumption. No dismissal was strengthened without executed evidence, and no finding was introduced because no axis produced attacker-reachable behaviour.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 2 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Checksum-before-use, zip-slip guard, TLS verification** — these protections live in `Mix.Utils.read_path`/`checksum` (extraction only on `{:ok, binary}`), `Archive.Install.validate_archive_path!`, and `read_httpc` (`verify: :verify_peer` + hostname check). This file invokes them correctly; the logic itself belongs to those files' reviews.

- **URL/SSRF injection via CSV version fields** (`run_install`, lines 78–83) — `elixir_version`, `hex_version`, `otp_release` come from the network CSV and are `String.replace`d into the archive URL. They land strictly in the *path*, after the `https://builds.hex.pm` authority is already written, so they cannot change host or scheme. Verified: with `x@evil.com`, `../../../..`, `foo/@evil.com`, and `://evil.com/x` as version values, `URI.parse/1` still reports `host="builds.hex.pm"`, `scheme="https"`. Any manipulation is additionally checksum-gated.
