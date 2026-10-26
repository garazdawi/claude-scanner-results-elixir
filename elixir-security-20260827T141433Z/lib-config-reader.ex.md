# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/config/reader.ex
<!-- sha256: 86a171aff0d1fe31f127fd2942a7ca6c8010b1931116b19c3abe997ba0a499c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

# Security review — `lib/elixir/lib/config/reader.ex`

<!-- sha256: 86a171aff0d1fe31f127fd2942a7ca6c8010b1931116b19c3abe997ba0a499c9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary
`Config.Reader` is a thin, faithful wrapper whose only powerful behavior — evaluating `.exs` config files as Elixir code — is its documented purpose and the caller's contractual responsibility. In the first pass I hedged that the actual `Code.eval_string` sink, the import-recursion guard, and the merge-into-`Application` sink lived in `config.ex`/`provider.ex` and were "out of scope for this file." I have now read both neighbour modules and traced the full call graph. The hedges resolve to a benign assessment: the downstream sinks behave exactly as assumed, and no actionable security defect is reachable through this file's public API.

## Security

**No findings.**

The six public entry points (`init/1`, `load/2`, `eval!/3`, `read!/2`, `read_imports!/2`, `merge/2`) were traced to their real sinks in the neighbour modules:

- `eval!/3`, `read!/2`, `read_imports!/2` → `Config.__eval__!` (`config.ex:319`) → `import_config!` (`config.ex:345`) → `Code.eval_string(contents, [], file: file)` (`config.ex:365`). **Confirmed by reading `config.ex`.** This is arbitrary code execution and is the documented function of the module (config = code). Reaching it requires the caller to hand the reader untrusted `.exs` content, which the review scope excludes as a finding about this repository.
- `merge/2` → `Config.__merge__` (`config.ex:369`) → `Keyword.merge` with `deep_merge/3` (`config.ex:375`). **Confirmed by reading:** `deep_merge` recurses only when *both* sides satisfy `Keyword.keyword?`; otherwise it returns `value2` verbatim. No `String.to_atom`, `apply`, or dispatch. The sink that acts on merged data — `Application.put_all_env(config, persistent: true)` — lives at `provider.ex:278`, not in this file.

### Ruled out

- **Arbitrary code execution via `read!/2` / `eval!/3` / `read_imports!/2`** — *Basis: verified.* First-pass execution evidence stands (`Config.Reader.eval!` printed `SIDE_EFFECT: ~c"140"` and read `System.get_env("HOME")`). Re-review adds the source-level confirmation of the sink: `import_config!/3` unconditionally calls `Code.eval_string(contents, [], file: file)` at `config.ex:365` on every path (both when imports are permitted and when the recursion/disabled guards fall through to the trailing eval). The docstrings say "Evaluates"/"Reads the configuration file" — no false safety claim. This is the module's documented purpose and fails the review's own exclusion. **Confidence: High.**
- **`merge/2` handling untrusted data (atom exhaustion / dispatch)** — *Basis: verified.* First-pass reproduction (map values carried through verbatim, no atomization) stands and is now corroborated at source: `config.ex:369-381`. Untrusted leaf/map values are inert — `deep_merge` never recurses into a non-keyword-list (a decoded-JSON map fails `Keyword.keyword?` and is returned as-is), and there is no atom creation anywhere in the merge path. The acting sink `Application.put_all_env/2` is at `provider.ex:278`. **Confidence: High.**
- **`merge/2` deep-recursion DoS** — *Basis: verified.* First-pass 200 000-level nested-keyword probe completed. Source confirms the only recursion is `deep_merge/3` self-calls bounded by input nesting depth, and it fires only when *both* sides are keyword lists at that depth — i.e. the attacker must supply trusted keyword *structure*, which the provider data-flow (JSON leaves merged under an operator-authored keyword skeleton, `provider.ex:73-79`) does not permit. **Confidence: High.**
- **Import-chain infinite recursion / re-entrancy** *(upgraded from a "Risks/caveats" hedge in first pass — previously deferred as out of scope)* — *Basis: verified by reading.* `import_config!/3` guards against a file importing itself transitively: `file in current_imports -> raise ArgumentError "attempting to load configuration ... recursively"` (`config.ex:356`), with the imports accumulator maintained in the process dictionary and reset/restored by `__eval__!`'s `try/after` (`config.ex:324-342`). A cyclic `import_config` chain raises rather than looping. Not a defect. **Confidence: Medium** (guard verified by source; I did not construct a live cyclic-import repro this pass).
- **Path traversal in `read!/2` / `eval!/3`** — *Basis: reasoned + source-confirmed.* `read!/2`/`eval!/3` call `Path.expand(file)` (`reader.ex:80,101,119`) with no `Path.safe_relative` confinement, but there is no root the path is meant to stay within — the caller explicitly names the file. `import_config/1` expands relative imports against `__DIR__` (`config.ex:306`), again with no intended jail. "Read the file the caller named" is not CWE-22; content trust is the caller's. **Confidence: Medium.** *Not executed:* I did not push a `../`-bearing path through `Path.expand` to observe the expanded result, so the "no confinement is intended" conclusion rests on the absence of any documented root, not on a differential run.
- **`load/2` → `resolve_config_path!({:system, name, path})`** — *Basis: reasoned + source-confirmed.* `provider.ex:199` computes `System.fetch_env!(name) <> path`. Both `name` and `path` originate from the release's `config_providers` list, validated at assembly time by `validate_config_path!/1` (`provider.ex:173-191`, requires `is_binary(name)`/`is_binary(path)`) and resolved at boot. An attacker who controls those already controls the release image. Confirmed no attacker-reachable injection into the env-var name. **Confidence: Medium.** *Not executed:* I did not run a `{:system, ...}` resolution with an adversarial `RELEASE_ROOT` value to inspect the concatenation, so unusual-byte/absolute-path interactions in the `<>` join stay unexamined.
- **`init/1` `Keyword.pop!(opts, :path)` raising on missing `:path`** — *Basis: reasoned.* `reader.ex:57` raises `KeyError` on operator-supplied provider options at assembly/boot; not attacker-influenced, no security impact. **Confidence: High.**

## Risks / caveats
- The first-pass caveat "I reviewed only `reader.ex`; the eval, the import-recursion guard, and the merge-into-`Application` sink are out of scope" is **now discharged**: `config.ex` and `provider.ex` were read this pass. The `Code.eval_string` sink (`config.ex:365`), the recursion guard (`config.ex:356`), and the `Application.put_all_env` sink (`provider.ex:278`) all behave as the first pass assumed.
- The whole assessment rests on the threat model that a project's own `.exs` config is trusted — the correct and universal Elixir assumption. An application that deliberately feeds *attacker-supplied* text to `Config.Reader.eval!/3` as a data parser gets arbitrary code execution, but that is documented-against caller misuse, not a defect here.
- Two ruled-out entries (path traversal, `{:system, …}` resolution) retain a *reasoned* basis with an explicit un-executed axis noted above. I did **not** upgrade them to findings because the code confirms no confinement root is intended and no attacker-reachable input feeds the concatenation; I did **not** upgrade them to *verified* because I ran no differential/adversarial input through `Path.expand` or the `<>` join this pass. They remain low-severity dismissals a later pass may re-examine.

## Did I verify this?
- **Read this pass:** `reader.ex` (full), `config.ex` (full — `__eval__!`, `import_config!`, `__merge__`, `deep_merge`, `validate!`), `provider.ex` (full — `validate_config_path!`, `resolve_config_path!`, `boot_providers`, `Application.put_all_env`).
- **Executed evidence (carried from first pass, still valid — same repo build `elixir 1.21.0-dev`):** live `eval!/3` code-execution repro; `merge/2` value-inertness repro; 200 k-deep merge recursion survival.
- **Confirmed by source-read this pass (no new execution):** the `Code.eval_string` sink is unconditional on all `import_config!` branches; `deep_merge` cannot recurse into decoded-JSON maps; the import-recursion guard raises on cycles; `{:system,…}` name/path are assembly-validated binaries.
- **Not executed this pass (axes left open, entries stay reasoned):** adversarial `../` path through `Path.expand`; adversarial `RELEASE_ROOT` through the `<>` concatenation; a live cyclic-`import_config` repro.

## Dropped in re-review
None. Every finding and ruled-out entry from the first pass is preserved; the "out of scope for this file" hedges were resolved by reading the neighbour modules rather than dropped, and the import-recursion guard was upgraded from a first-pass caveat to an explicit source-verified ruled-out entry.
