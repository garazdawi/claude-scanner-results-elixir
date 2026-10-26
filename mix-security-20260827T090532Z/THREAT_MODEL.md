```markdown
# Threat Model: `elixir/lib/mix` (Elixir's Mix build tool)

## Protocol / standard implemented
This is **Mix**, Elixir's build tool and package/dependency manager — not a network protocol implementation. Its security-relevant behaviour is that of a **build system + package installer** that fetches remote code, extracts archives, verifies checksums, and executes downstream build scripts. Relevant standards/conventions: HTTP/HTTPS + TLS server-cert verification (via OTP `:httpc`/`:ssl`/`:public_key`, RFC 5280 path validation, RFC 6125 hostname matching); SHA-512 checksums; ZIP (`.ez` archives via `:zip`) and tarball formats; Semantic Versioning (`Version`); Git transport (via the `git` CLI).

Documented public entry points are the `Mix.Tasks.*` modules (each a `run/1` behaviour) — notably `archive.install`, `archive.build`, `escript.install`, `local.hex`, `local.rebar`, `deps.get`, `deps.compile`, `deps.update`, `deps.unlock`, `compile.*`, `escript.build`, `release`, `run`, `eval`, `cmd`, `new`. Library APIs: `Mix`, `Mix.Local`, `Mix.Utils` (`read_path/2`, `read_httpc/1`, checksum), `Mix.SCM`/`Mix.SCM.Git`/`Mix.SCM.Path`, `Mix.Dep.*`, `Mix.Project`.

## Trust / contract boundaries
- **Network peer (S3/Hex/Git/arbitrary URL):** `Mix.Utils.read_path/2` → `read_httpc/1` fetches attacker-influenceable bytes. URL, redirect `Location`, response body, and CSV metadata (`Mix.Local.parse_csv`/`find_matching_versions!`) are peer-controlled. Peer-controlled `.ez`/tarball **file names inside archives** are the classic sink.
- **Untrusted file (the archive/tarball itself):** `archive.install`'s `install/3` and `validate_archive_path!/2` treat every zip entry path as hostile — reject absolute paths, `..`, backslash, NUL, and entries escaping a single archive root. This is the v1.20.1 path-traversal hardening; it is the **primary safety invariant** of this folder.
- **Operator-supplied config:** `mix.exs` (`Mix.Project.config`), `config/*.exs`, `mix.lock`, `deps`, env vars (`MIX_PATH`, `HEX_CACERTS_PATH`, `HTTP(S)_PROXY`, `NO_PROXY`). Note `mix.exs`/config are **executed as code** — an operator who controls them already has full RCE; that is by design, not a boundary.
- **Contracts callers must honour:** checksum verification only applies to the `--sha512` path for local installs (Git/GitHub/Hex builds run source code and are *not* checksum-gated — documented in the "Security" section). `check_install_spec` enforces `.ez` extension. `find_by_elixir_version` assumes CSV columns are well-formed (`hd/1`, list destructuring can crash on malformed metadata).

## Attacker model
- **In scope:** a **malicious/compromised remote source or MITM** serving a crafted archive to `archive.install`/`escript.install`/`local.*`; a crafted `.ez` attempting zip-slip file writes outside the archive dir; malformed CSV/metadata causing crashes. TLS is `verify: :verify_peer` with hostname checking, so passive MITM is out of scope unless the operator overrides CA via `HEX_CACERTS_PATH`.
- **Out of scope (by explicit design):** the operator themselves. `mix.exs`, dependency build scripts, and installed archives **run arbitrary code** — the docs state "Archives must be installed only from sources you trust" and that an installed archive loads as a plugin on *any* Mix command. So "installing evil code executes it" is not a bug here; the defended boundary is narrower: an archive must not write **outside its extraction directory** or crash the tool, and remote transport must be authenticated.

## Common attack patterns for this family
- **Zip-slip / tar-slip path traversal** (`../`, absolute paths, symlinks, NUL truncation, backslash on Windows) — the marquee risk; cf. CVE-2025-4748 in OTP `:zip`. Review `validate_archive_path!` and any `:zip.extract`/`:erl_tar` call for gaps (e.g. symlink entries, case-folding, Unicode normalization).
- **Checksum/signature bypass** — SHA-512 optional and local-only; a source-build path has no integrity check.
- **Dependency confusion / lockfile tampering** — `mix.lock`, `deps.get`, `remote_converger`, Git ref pinning.
- **TLS downgrade / cert-verification disable** — check the `middlebox_comp_mode`/IP-fallback retry branches don't silently weaken verification; `HEX_CACERTS_PATH` override.
- **Git command-argument injection** — `Mix.SCM.Git` shells out via `System.cmd("git", args)` with dep-supplied refs/URLs.
- **Metadata parser crashes / DoS** on malformed CSV or archive listings.

## Supervisor / process structure
Long-lived global state processes (started under Mix's own supervision tree, not per-request): `Mix.State` (GenServer + public ETS for builtin apps/config), `Mix.ProjectStack` (GenServer holding the project/config stack), `Mix.TasksServer` (Agent + named public ETS tracking which tasks have run — the idempotency guard), `Mix.PubSub`/`sync/pubsub` + `pubsub/subscriber` (compile invalidation events). These are **trusted local infrastructure**; the attacker is remote content, not another BEAM process. Crashing any of them aborts the build but does not cross a privilege boundary. Note the ETS tables are `:public` — any co-resident code can mutate task/idempotency state, but such code is already operator-trusted.

## Cross-file invariants
- **`archive.install`'s `validate_archive_path!` is the single choke point** protecting `:zip.extract` in `install/3` — a reviewer must not assume `:zip.extract` is independently safe; it relies entirely on prior path validation. Any *other* extraction site in the folder (escript, deps) needs its own equivalent check.
- **`Mix.Utils.read_path` centralises all remote fetching**; per-file reviewers seeing a raw URL should trace back to confirm it flows through `read_httpc` (TLS-verified) and not a bespoke client.
- **`Mix.TasksServer` guarantees each task runs once** — code assuming a task's side effects already happened depends on this ETS state, not on call order.
- **`check_install_spec` (`.ez` extension) runs before `install/3`** — extension checks are not repeated downstream.
- **Checksum verification is caller-opt-in**, so downstream code must not assume fetched bytes are integrity-verified unless `--sha512` was supplied.

Sources: [Elixir v1.20.1 release](https://elixirforum.com/t/elixir-v1-20-1-released/75607), [OSV CVE-2024-26256](https://osv.dev/CVE-2024-26256), [JFrog Zip Slip](https://research.jfrog.com/vulnerabilities/archiver-zip-slip/), [Paraxial Hex security](https://paraxial.io/blog/hex-security)
```