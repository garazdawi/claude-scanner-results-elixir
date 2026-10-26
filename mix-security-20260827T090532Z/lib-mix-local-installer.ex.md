# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/local/installer.ex
<!-- sha256: d1ffabbbeb90dcfc56fc550707a9f6d96d78c75090c5c738fe5b73bf930bc8ab -->

_Analysed on 2026-08-27T09:48:25Z_

I have enough to conclude. Let me confirm the final piece — that the callers feed operator command-line argv, and that the one place peer data reaches this file's `Mix.raise` (the `{:remote, message}`) is `inspect`-escaped (no raw peer bytes / terminal-escape injection).

Both are now confirmed: `httpc_response/1` builds `:remote` messages from an integer `status` or `inspect(reason)` (utils.ex:940-944), and `read_path` runs `checksum/2` before returning `{:ok, binary}` (utils.ex:809/822), so the sha512 is verified before the payload is used.

## Summary

`Mix.Local.Installer` is operator-driven glue for `mix archive.install`/`escript.install`/`*.uninstall`; every sink in this file consumes operator-supplied command-line arguments, and the genuine attacker-controlled bytes (fetched archive, git ref, HTTP redirect) are passed straight through to sinks that live in collaborating files, so no attacker-data-crosses-a-boundary finding lands in this file.

## Security

No findings.

### Ruled out

- **`String.to_unsafe_atom/1` atom creation (lines 255, 259, 286)** — this is a real unbounded atom-creation sink (verified: `String.to_unsafe_atom("hello world, evil")` returns `:"hello world, evil"`), but `package_name`/`app_name` originate only from `parse_args`, which is reached exclusively from `Mix.Tasks.{Archive,Escript}.Install.run/1` passing the task's command-line `argv` (archive.install.ex:91). The bytes are the operator's own, not a dependency/network/archive/git peer, so the CWE-770 angle has no second party. Out of scope (operator-controlled).
- **`uninstall/4` → `File.rm_rf!(found)` with `Path.join(root, name)` (lines 306-315)** — `name = List.first(argv)` is a command-line argument; `..` in it traverses out of `root`, but the person typing `mix archive.uninstall ../…` is the operator deleting a path they named. No external party's data reaches `name`. Out of scope (operator self-inflicted).
- **`maybe_sparse_dir` → `Path.join` into `package_path` (lines 413-424)** — `opts[:sparse]` is the operator's `--sparse` flag; traversal is operator-controlled.
- **`inspect(dep_spec)` interpolated into the generated `mix.exs` then compiled (lines 364-376)** — a code-generation site, but `dep_spec` holds only atoms/strings/booleans built in `parse_args`, and `inspect/1` emits properly-escaped Elixir literals, so no breakout; inputs are operator command-line values regardless.
- **git ref / URL argument injection (`git_fetcher`, `ref_to_config`, lines 224-295)** — `url`/`branch`/`tag`/`ref`/`sparse` flow as a structured keyword list into `dep_spec` and ultimately to `Mix.SCM.Git`'s `System.cmd("git", argv)` (a different file); nothing here shells out or interpolates them into a command string, and they are operator-supplied.
- **Checksum bypass / TOCTOU on `--sha512`** — `install/3` forwards `opts` to `Mix.Utils.read_path`, which runs `checksum/2` and only yields `{:ok, binary}` on a match before `module.install` uses the bytes (utils.ex:803-848). Verified the ordering: checksum is enforced before payload use, and `--sha512` is explicitly rejected for the `:fetcher` (git/hex) path (lines 87-90). No bypass.
- **Terminal-escape / log injection via the `{:remote, message}` heredoc (lines 142-171)** — the only peer-influenced value reaching `Mix.raise` here is `message` from `read_path`, which for remote errors is `"…failed with: {:bad_status_code, <int status>}"` or `"…failed with: #{inspect(reason)}"` (utils.ex:940-944). The status is an integer and `reason` is `inspect`-escaped, so no raw peer bytes survive into the terminal. `src`/`basename` in the same heredoc are the operator's own URL.
- **`basename = Path.basename(URI.parse(src).path)` (line 128)** — `Path.basename` returns only the final segment (no `/`), so it cannot re-introduce traversal, and an HTTP redirect cannot alter it because it is derived from the original operator `src`, not the redirect target. A path-less URL (`https://host`) makes `URI.parse(...).path` `nil` and crashes `Path.basename`, but that is an operator-supplied URL crashing the operator's own invocation — not security-relevant.

## Risks / caveats

- **Predictable fetcher temp directory (lines 432-435, `tmp_path`)** — the name uses only `:crypto.strong_rand_bytes(4)` (32 bits) and is created with `File.mkdir_p!`, which succeeds on a pre-existing directory/symlink; a *local* co-tenant on a shared `/tmp` could in principle pre-plant a symlink and win the race so `File.write!(…/mix.exs)` / the build operate through it. I did not report this because it requires a local BEAM-external attacker, which is outside this folder's stated attacker model (remote/MITM content), and the install already builds arbitrary operator-chosen code; it is at most a defense-in-depth Low. Safer form: a higher-entropy name plus a create-exclusive (fail-if-exists) directory.
- **`file_url?` accepts `http://` as well as `https://` (lines 123-125)** — an operator who passes a plaintext `http://` URL without `--sha512` is exposed to a network MITM substituting the archive (which, once installed, loads on every subsequent `mix` command). This is operator-chosen transport and the actual fetch/verify logic lives in `Mix.Utils.read_httpc` (out of this file), so I did not score it here, but the routing predicate is the gate that permits it.
- **Out-of-file sinks I did not review** — the archive/escript extraction (`module.install` → `:zip.extract`/`validate_archive_path!` in `Mix.Tasks.Archive.Install`), the actual TLS/redirect handling and CA override (`Mix.Utils.read_httpc`, incl. `HEX_CACERTS_PATH` and the `middlebox_comp_mode`/IP-family fallback retries), and git argument handling (`Mix.SCM.Git`) are where the real attacker-controlled bytes terminate; they are in other files and were only traced far enough to confirm this file forwards data to them rather than sinking it itself.
- All behavioural checks were run against the in-tree build (`String.to_unsafe_atom` atom creation confirmed; `read_path`/`httpc_response` ordering and message shapes read directly). I did not execute a full end-to-end `mix archive.install` against a live malicious server, as the in-file conclusions did not depend on it.

