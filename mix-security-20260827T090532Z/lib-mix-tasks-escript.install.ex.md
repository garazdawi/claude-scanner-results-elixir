# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.install.ex
<!-- sha256: 10a3f8da47059eb866f771cb4235333d765cc8f11a8baad1bd1bb1c435617f93 -->

_Analysed on 2026-08-27T11:05:26Z_

I have completed the trace and verification. Every sink in this file is either confined upstream, escaped, or by-design; the genuinely dangerous surfaces live in sibling files. Here is the report.

## Summary
`mix escript.install`'s task module is a thin `Mix.Local.Installer` callback implementation whose only file-writing sink is confined to `~/.mix/escripts` by an upstream `Path.basename`, whose warning strings are `inspect`-escaped, and whose network/checksum/atom/git-argument surfaces all live in sibling files — no security vulnerability is reachable through its documented public API.

## Security

No findings.

### Ruled out

- **Path traversal in `destination/1` → `install/3` file writes (`escript.install.ex:143`, `File.write!/File.rm/File.mkdir_p!`)** — `dst = Path.join(Mix.path_for(:escripts), basename)` uses an externally-derived `basename`, but the only public entry point (`run/1` → `Mix.Local.Installer.install` → `local_install`, `installer.ex:128`) sets `basename = Path.basename(URI.parse(src).path)` *before* calling `install/3`. `Path.basename` collapses any component to a single path segment (verified: `.../etc/cron.d/x`→`"x"`, percent-encoded `%2e%2e%2f…` stays literal, `passwd`, etc.), so no `/` can survive and multi-level `..` is impossible. The sole one-level escape (`basename == ".."` or `""`) targets a directory and makes `File.write!` raise `:eisdir` — verified nothing lands outside the escripts dir (`root contents: ["escripts"]`). Reaching `install/3` with a raw multi-segment `basename` would require a caller violating the documented `basename :: String.t()` callback contract, which is out of scope. Defense-in-depth only: `destination/1` could assert the component has no separators or use `Path.safe_relative/1`.
- **Symlink/TOCTOU overwrite via pre-created `dst` (`escript.install.ex:114-121`)** — `File.rm(dst)` runs before `File.write!(dst, binary)`, removing any attacker-planted symlink (the link, not its target) so the subsequent write creates a fresh regular file; the target dir is the user's own `~/.mix/escripts`, so any racer already has the victim's privileges.
- **Terminal-escape / metacharacter injection in `check_discoverability/3` and the install info line (`escript.install.ex:125,166-184`)** — the warning strings wrap values in `inspect(...)`, which escapes control bytes (verified: `"\e[31m…"` → `"\"\\e[31m…\""`); the non-`inspect` `Path.relative_to_cwd(dst)` value is derived from the operator-typed URL/path (or a malicious dependency name, which already holds RCE-by-design during `build/2`), not from a network peer.
- **`escript?/1` weak format check (`escript.install.ex:192-195`)** — it only sanity-checks shebang + `PK\x03\x04` magic; it is not a trust boundary (a malicious escript passes trivially and escripts are documented as code you must trust). `String.split(binary, "\n", parts: 4)` handles the zip/invalid-UTF-8 payload without crashing (verified) and gates the writes, so a non-escript URL is rejected before any file is written.
- **`--sha512` "local path only" doc note (`escript.install.ex:53-54`)** — not a checksum bypass: `Mix.Utils.read_path` applies `checksum/2` to the URL branch as well as the local branch (`utils.ex:809,822`) and *before* returning the bytes to `install/3`, so a URL install with `--sha512` is in fact verified; the doc is merely conservative. Escript source-builds (git/hex) explicitly reject `--sha512` (`installer.ex:87-90`) — expected, since those run code, not a fixed artifact.
- **`check_install_spec/2` always returns `:ok`, including the redundant `{:url, _}` clause (`escript.install.ex:97-101`)** — no security consequence: escripts have no required extension (unlike archive `.ez`), and `install/3`'s `escript?` gate still rejects non-escript payloads. Redundant-clause code smell only.
- **`build/2` executes fetched dependency code (`escript.install.ex:134-138`)** — runs `loadconfig`/`escript.build` in the fetched project; this is the documented behaviour ("executes code from the source during installation") and the operator-trust model, not a defect.

## Risks / caveats
- **Adjacent sinks are out-of-file and were not scored here** but are reachable from this task's `run/1`: `String.to_unsafe_atom/1` on the operator-supplied Hex package/app name and git app name (`installer.ex:255-259,286`), and git ref/URL argument handling that flows into `Mix.SCM.Git` (`scm/git.ex`) via the `{:fetcher, dep_spec}` built in `installer.ex`. These are the git-arg-injection and atom-exhaustion surfaces the escript task inherits; they belong to those files' reviews. The inputs there are operator CLI arguments (not a remote peer), which lowers their severity.
- **Network path (`utils.ex:read_httpc`) reviewed only enough to clear this file's URL flow**: TLS is `verify_peer` with `pkix_verify_hostname_match_fun(:https)`, the `HEX_CACERTS_PATH` override is operator-controlled, and the two retry branches (`ipfamily` fallback; `middlebox_comp_mode: false`) do not disable peer/hostname verification. A full audit of `read_httpc` redirect handling and `proxy_config` belongs to the `utils.ex` review.
- I did not exercise a full end-to-end `mix escript.install <url>` against a live server (no network in-sandbox); the file-local claims (basename confinement, `:eisdir` on `..`, `inspect` escaping, `escript?` on binary payloads) were reproduced directly with the built-from-source `1.21.0-dev` toolchain.

