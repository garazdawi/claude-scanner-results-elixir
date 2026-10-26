# Cluster cross-file findings

No cluster cross-file findings

---

**Scope analyzed** (each source file read from disk; shared out-of-cluster sinks/definitions resolved and reasoned from real code):

- **Checksum/fetch chain — consistent and correct.** Both `local.hex.ex:88` (`Archive.Install.run([url, "--sha512", sha512 | argv])`) and `local.rebar.ex:126` (`install_from_path(manager, url, Keyword.put(opts, :sha512, sha512))`) feed the CSV-derived SHA-512 into the same verifier. I confirmed the URL path is actually checked: `Mix.Local.Installer.install/3` routes both `:local` and `:url` specs to `local_install` (`installer.ex:97`), which calls `Mix.Utils.read_path(src, opts)`, and `read_path` runs `checksum/1` on both the `url?` and `file?` branches (`utils.ex:803-828`, `832-848`). The `archive.install` moduledoc claim that `--sha512` "Only applies to installations via a local path" (`archive.install.ex:54-55`) *understates* the protection — the code verifies the URL download too — so it is not a hole. No validation split between the two installers.
- **No nil-digest bypass reachable from the cluster.** `checksum/1` skips only when `opts[:sha512] == nil`; `find_by_elixir_version` (`local.ex:219-228`) always returns CSV column 1 as a binary (worst case `""`, which fails closed). Neither `local.hex` nor `local.rebar` can hand the verifier a `nil`.
- **Remote taint is authenticated.** The only attacker-influenceable bytes (CSV metadata + archive/escript) arrive over `read_httpc` with `verify: :verify_peer` + hostname match (`utils.ex:888-892`); the TLS retry branches keep `verify_peer` (no downgrade split). A malicious *mirror* controls both artifact and checksum by design — checksum-from-same-source is not a cross-file defect. The known malformed-CSV `hd([])` crash lives entirely in `local.ex` (out of cluster) and is already noted in the folder threat model; it is not a cluster cross-file issue.
- **Profilers are independent, identically-shaped developer-CLI surfaces.** `profile.{cprof,eprof,fprof,tprof}.ex` do not call one another; each funnels developer-supplied code through `Mix.Tasks.Run.run` + `Code.string_to_quoted!`/`compile_quoted` (== `mix run`). The one genuine cross-file contract — `cli.ex:30` `Tprof.parse!(MIX_PROFILE_FLAGS)` → `Mix.State` → `task.ex:533` `Tprof.profile` — is driven by operator env vars and the atom-keyed opts contract holds end to end.
- **`new.ex` / `release.ex` operate on operator/developer-trusted input.** `new.ex:212` `String.to_unsafe_atom` is gated within the same file by the `^[a-z][a-z0-9_]*$` regex (`invalid_app` short-circuits before `reserved_app`); `release.ex:1071` converts the operator-typed release NAME. Neither crosses an untrusted boundary reachable from another cluster file.

No trust-boundary validation split, cross-file taint-to-sink, authn/authz split, broken "caller checks" precondition, or crypto/secret mishandling spans two or more files in this cluster.
