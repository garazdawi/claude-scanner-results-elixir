# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/lock.ex
<!-- sha256: 8681dfa9190e722a7a01b01abff2e80c31e775e45d091addb85d720ce937ce8e -->

_Analysed on 2026-08-27T09:38:35Z_

Verified. `inspect` escapes lock values, so network-supplied checksums/versions round-trip as plain string literals — the `write/2` value path cannot inject code on rewrite. I have what I need.

## Summary

`Mix.Dep.Lock.read/1` parses and then **evaluates** the `mix.lock` file with `Code.eval_quoted/3`, so any Mix task that reads the lockfile executes arbitrary code embedded in that data-looking file at build time.

## Security

#### `CSCAN-lock-lockfile-code-eval`
- **Finding ID** — `CSCAN-lock-lockfile-code-eval`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `lib/mix/lib/mix/dep/lock.ex:31`, `read/1` (the `with` chain, `Code.eval_quoted(quoted, [], opts)`)
- **Class** — Code execution (arbitrary code from a data file, on the build path)
- **Impact** — A `mix.lock` file is semantically *data* (a map of dependency hashes) but is executed as Elixir code. Any `mix.lock` that reaches a victim yields arbitrary code execution the first time the victim runs `mix deps.get`, `mix deps.update`, `mix deps.unlock`, `mix compile`, or `Mix.install(lockfile: …)` — all of which call `Mix.Dep.Lock.read()` (confirmed call sites: `mix.ex:1005-1006`, `compilers/elixir.ex:314`, `dep/converger.ex:88`, `tasks/deps.*`). The payload runs before compilation, with the developer's full privileges, and the eval still returns a normal-looking map so nothing appears wrong. The escalation over the "`mix.exs` runs anyway" baseline is the **partial-trust** case where the attacker controls only the lockfile, not the project's trusted `mix.exs`: a dependency-bump PR (or an automated lockfile-update bot) that touches *only* `mix.lock` looks like hash noise in review, but checking out the branch and running any Mix task detonates it. The same applies to `Mix.install(lockfile: <shared/downloaded path>)`, which reads a foreign lockfile via this function.
- **Evidence** — `verified`. Crafted `evil.lock`:
```elixir
%{ "innocent_looking": (File.write!("/tmp/lockeval/PWNED", "code ran ..."); {:hex, :innocent_looking, "1.0.0"}) }
```
Ran `elixir -e 'Mix.Dep.Lock.read("/tmp/lockeval/evil.lock")'` → output `read result: %{innocent_looking: {:hex, :innocent_looking, "1.0.0"}}` **and** the side-effect file `/tmp/lockeval/PWNED` was created (`-rw-r--r-- … /tmp/lockeval/PWNED`, body `code ran at nil`). The embedded expression executed during `read`, and the function returned a clean map.
- **Suggested fix** — Do not `Code.eval_quoted/3` the lockfile. After `Code.string_to_quoted/2`, walk the quoted form and accept only literal data (a map whose keys are binaries/atoms and whose values are nested tuples/lists/binaries/atoms/numbers), rejecting any AST node that is a function call/operator/`unquote`/`__aliases__`-dispatch; then materialise the literal (e.g. via `Code.eval_quoted` only *after* the node has been proven call-free, or by building the term directly from the validated AST). This preserves the existing on-disk format while removing the execution primitive.

### Ruled out
- **`write/2` value interpolation (`inspect(rev, limit: :infinity)`, line 54)** — the lock *values* (`rev`) originate from dependency resolution / the Hex remote converger (network). `inspect` canonicalises and escapes them, so a hostile checksum/version string round-trips back as a plain string literal, not code (verified: a `rev` string containing `") ; System.cmd(...)` re-parsed to the identical binary). No injection on the write→read cycle via values. `limit: :infinity` correctly avoids truncation that would otherwise break round-trip integrity.
- **`write/2` key interpolation (`~s(  "#{app}": …)`, line 54)** — the key `app` is *not* escaped, so an atom app-name containing `"` would corrupt the emitted line. Not reported as a finding: `app` is the OTP application name sourced from operator-declared `deps` in `mix.exs` (or Hex package names constrained to `[a-z0-9_]`), i.e. it does not cross a network/dependency trust boundary through this file's API, and driving a hostile key would require the caller to pass a map violating the resolved-lock contract. Worth hardening (quote the key with `inspect` too) as defense-in-depth, but not a boundary-crossing bug here.
- **`assert_no_merge_conflicts_in_lockfile/2` (lines 68-75)** — a data-integrity/UX aid that raises on git conflict markers; it is not a security control and its bare-expression placement in the `with` behaves correctly (raises propagate past `else`, which only catches non-matching values). It does not gate the eval, so its presence/absence does not change the finding above.

## Risks / caveats
- **Trust-boundary judgement, not mechanism, is the soft part.** The code-execution mechanism is verified beyond doubt. The severity hinges on whether one accepts "lockfile eval" as accepted Mix design (the folder threat model lists `mix.lock` under operator-supplied config, though its "executed as code by design" carve-out names only `mix.exs`/`config`). In the fully-untrusted-repo scenario `mix.exs` executes regardless, so the lockfile eval adds nothing *there*; I rated it Medium on the strength of the distinct partial-trust vectors (lockfile-only PRs, update bots, `Mix.install` lockfile reuse) where the lockfile is the *sole* attacker-controlled code path. A reviewer prioritising the build-path "weaker expectations / wider reach" guidance could justifiably call this High; one treating it as long-standing accepted behaviour could call it Low.
- **`external_lockfile` reach not exercised end-to-end.** I confirmed by reading `mix.ex:972-1007` that `Mix.install(lockfile: …)` funnels a foreign lockfile into `Mix.Dep.Lock.read/1`, but I did not run a full `Mix.install` to detonate it that way; the direct `read/1` repro already proves the sink, and that path uses the same function.
- **DoS sub-cases folded in.** A malicious lockfile can also hang or exhaust memory (infinite loop / huge term while still returning a map). These are strictly weaker than the RCE and are not separately itemised.
- I reviewed only this file; the provenance of `write/2`'s `map` keys was reasoned about from call sites (`Mix.Dep.Fetcher`) rather than audited in depth, which is why the key-interpolation issue is left as a caveat rather than a finding.

