# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/config/provider.ex
<!-- sha256: 51b99db9cb90130e2c48b766722a74de24344e954ffd6fba0fedda6a6aa89152 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:35:38Z_

My review is complete. All potential sinks traced and behaviors verified by running against the built-from-source Elixir. Here is the report.

---

## Summary

`Config.Provider` is release-boot configuration machinery whose every input (config-path components, provider modules, config-file contents, `extra_config`) is authored in the developer's release configuration, so none of its sinks are reachable by untrusted data through its documented public API — no meaningful security findings.

## Security

`No findings`

The documented public API is `validate_config_path!/1` and `resolve_config_path!/1` (plus the `init/1`/`load/2` behaviour callbacks that *implementers* provide). Both public functions are pure/near-pure and their inputs are release-author configuration. Every dangerous-shaped sink (`:file.consult`, `File.rm`, `File.write`, `provider.init`/`provider.load` dispatch, `:init.restart`) lives in the `@doc false` boot flow and operates exclusively on developer-authored release data. There is no path by which network data, an untrusted archive, a dependency's runtime data, or a user-supplied file reaches a sink here.

### Ruled out

- **Round-trip term injection in `write_config!`→`read_config!` (lines 422–435, 390–401).** `:io_lib.format("~tw", [config])` + `:file.consult` — a classic serialize/parse asymmetry site. **Verified safe:** hostile binary/atom values containing `.`, newlines, and `]. [evil:` are written as escaped literals and read back byte-identical as a single term (`{:ok, [[myapp: [a: "]. [evil: :haha"]]]}`); `~w` cannot be made to emit an extra top-level term. Basis: verified.
- **Path traversal via `validate_config_path!`/`resolve_config_path!` (lines 173–199).** **Verified** that `validate_config_path!({:system, "X", "/../../etc/passwd"})` returns `:ok` and `resolve_config_path!` yields `"/app/root/../../etc/passwd"` — no `..` confinement. Not a finding: the path component is set by the release author at assembly time (`init/3`) and the env-var name is release-author-chosen; the documented contract is literally "concatenation of the environment variable with the given path." No untrusted boundary is crossed. Basis: verified behavior + trust-boundary analysis.
- **Atom creation / atom-table exhaustion via `:file.consult` in `read_config!` (line 391).** **Verified** that `:file.consult` interns atoms from file contents (`brand_new_atom_qwerty` became resolvable via `list_to_existing_atom`). Not a finding: the consulted `path` is the release's own sys.config, resolved from a release-configured path — trusted, not attacker-supplied. Basis: verified.
- **`read_config!` crash on malformed config (lines 390–401).** **Verified** an empty file yields `{:ok, []}`, which matches neither `{:ok, [inner]}` nor `{:error, _}` → `CaseClauseError`, aborting boot. Not a security finding: input is the trusted release config and the outcome is fail-closed (boot aborts). Basis: verified.
- **Dynamic dispatch `provider.init(init)` / `provider.load(acc, state)` (lines 212, 406).** Dispatch targets come from the release's `config_providers` list, not from any string-to-atom conversion of external data (grep confirmed no `*.to_atom`/`Module.concat`/`apply` sinks). A malicious provider module already implies build/release-time code execution. Basis: verified (grep) + analysis.
- **`File.rm(path)` prune and `:init.restart` (lines 226, 375–382).** Reachable only through the internal `boot/0` flow; `path` is the trusted resolved config path and reboot mode comes from `:elixir` app env. No external influence. Basis: analysis.

## Risks / caveats

- **Secrets persisted to disk (design tradeoff, not a public-API finding).** When `reboot_system_after_config: true`, `write_config!` (lines 422–435) writes the *fully merged* runtime config — which may include secrets loaded by providers from `runtime.exs` — to the config file in plaintext. Pruning it afterward is opt-in: `prune_runtime_sys_config_after_boot` defaults to `false` (line 161), so by default the file with secrets remains on disk. This is inherent to the reboot-based mechanism and is documented, and reading it requires filesystem access to the release; I flag it as residual hardening risk rather than a vulnerability reachable via the documented API with untrusted data.
- **Not exercised end-to-end.** I could not drive the full `boot/0` orchestration in a real release environment (it needs `:init.restart`, release env vars, and a release layout); I verified its individual sinks (serialize/parse round-trip, `:file.consult`, path validation/resolution) as isolated components and reasoned about the orchestration by reading. The component behaviors above are `verified`; the assembled-flow reasoning is `asserted`.
- **Windows edge.** `validate_config_path!` gates on `Path.type(path) != :relative`; on Windows, `Path.type/1` returns `:volumerelative` for forms like `C:foo`, which would pass the "absolute path" gate. Untested (Linux sandbox) and the path is release-author-controlled, so it is a documentation/robustness nit at most.
- **Scope.** Findings were judged against the documented public API with untrusted data. Behaviors reachable only by a malicious *operator* who already controls the release image, environment variables, or config providers were treated as outside the boundary, consistent with this module's threat model (the developer/operator authors all inputs).

