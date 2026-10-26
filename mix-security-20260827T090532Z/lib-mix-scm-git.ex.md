# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/scm/git.ex
<!-- sha256: 4f180f510c899f96f6ecb84116fdc6a8a37a1681318eed5e99df2e53950bf8d7 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/scm/git.ex
<!-- sha256: 4f180f510c899f96f6ecb84116fdc6a8a37a1681318eed5e99df2e53950bf8d7 -->

_Re-reviewed on 2026-08-27T10:08:18Z (hedge-resolution pass)_

The two hedges the first-pass reviewer flagged were both about **where the locked revision is (or isn't) validated**: the within-file assertion that `get_lock_rev` never passes through `validate_refspec`, and the out-of-file caveat that validation "elsewhere in Mix" was not traced. I resolved both by reading: the entire `git.ex` data flow (below) and the two neighbour modules that transport the lock tuple from `mix.lock` into `dep.opts[:lock]` — `Mix.Dep.Lock` (`dep/lock.ex`) and `Mix.Dep.Converger.put_lock/2` (`dep/converger.ex:253`, with `Mix.Dep.Loader`). Neither neighbour validates the rev. The git *behaviour* underpinning the finding was already executed in the first pass and is unchanged; only the data-flow provenance moves from `asserted` to `verified`.

## Summary

`Mix.SCM.Git` (the git dependency fetcher run during `mix deps.get`/`deps.compile`) contains no independently-reachable high-severity vulnerability — every attacker-influenced input originates from `mix.exs` or `mix.lock`, both of which Mix evaluates as code upstream — but it has two verified defense defects worth a Low: a hook-disabling control that sets a literal `''` directory instead of disabling git hooks, and a locked-revision value that reaches `git fetch`/`git checkout` as a standalone argv element without the `-`-prefix argument-injection guard that `:branch`/`:ref`/`:tag` receive.

## Security

#### `CSCAN-git-hookspath-literal-quotes`
- **Finding ID** — `CSCAN-git-hookspath-literal-quotes`
- **Severity** — Low
- **Confidence** — high
- **Location** — `git.ex:111` (`checkout/1`) and `git.ex:145` (private `checkout/2`, submodule branch) — `~w[-c core.hooksPath='' ...]`
- **Class** — Dead / unreachable defensive code (ineffective security control)
- **Impact** — The `-c core.hooksPath=''` is intended to *disable* git hooks so a malicious dependency repo/submodule cannot run a hook on the developer's machine during `mix deps.get`. Because the value is written inside the `~w` word sigil, the two `'` characters are literal: git receives `core.hooksPath=''` and resolves the hooks directory to a directory **named `''`** (two single-quote chars) relative to the working tree — i.e. into checked-out, attacker-controlled content — rather than disabling hooks. For the `submodule update` command this points hooks at repo-controlled worktree content instead of the (empty) default, which is *strictly worse* than omitting the flag. The intended hardening therefore provides no protection.
- **Evidence** — `verified`. `git -c "core.hooksPath=''" rev-parse --git-path hooks` → `''`. Placing an executable `post-checkout` in a directory named `''` and running `git -c "core.hooksPath=''" checkout -b third` executed it (`HOOK_RAN_via_literal_quotes` created). I could **not** demonstrate execution through Mix's exact command sequence on git 2.47.3: `-c core.hooksPath=''` is not persisted by `init` (`.git/config` has no `hooksPath`; effective path stays `.git/hooks`), the main `git checkout --force` (line 142) has no hooksPath override and used the empty `.git/hooks`, and `submodule update --init --recursive` did not fire a tracked-content `''/post-checkout`. So this is a broken/misdirected control rather than a demonstrated RCE — but it is fragile and version-dependent.
- **Suggested fix** — Disable hooks with a value git honours as "no hooks", not a shell-style empty-quote literal: pass `-c` `core.hooksPath=/dev/null` (or an absolute path guaranteed empty) as a single argv element, e.g. `["-c", "core.hooksPath=/dev/null", ...]`, and apply it to *every* hook-running command (including the `git checkout` on line 142), not only `init`/`submodule update`.

#### `CSCAN-git-lock-rev-arg-injection`
- **Finding ID** — `CSCAN-git-lock-rev-arg-injection`
- **Severity** — Low
- **Confidence** — high
- **Location** — `git.ex:137` (`refspec_switch(opts, lock_rev || ...)` → `git fetch ... origin <rev>`) and `git.ex:142` (`git checkout --force --quiet <rev>`); source `get_lock_rev/2` at `git.ex:303`
- **Class** — Command execution (git argument injection)
- **Impact** — `validate_refspec/1` (lines 241–267) explicitly rejects a `:branch`/`:ref`/`:tag` that starts with `-` ("must not start with -"), demonstrating the authors treat a `-`-prefixed refspec as an argument-injection hazard. The revision taken from `mix.lock` (`get_lock_rev`) bypasses that guard entirely: it flows straight into `git fetch … origin <rev>` (when the dep also has `:tag`/`:branch`/`:depth`, so `refspec_switch` emits `["origin", rev]`) and into `git checkout --force --quiet <rev>`. Because `System.cmd` passes each list element as one argv entry and git permutes options after positionals, a lock rev of `--upload-pack=<cmd>` is parsed by `git fetch` as the `--upload-pack` option and executes `<cmd>` for a local/`file:`-transport dependency. Practical reachability is limited: `mix.lock` is itself evaluated by Mix (`dep/lock.ex` uses `Code.eval_quoted`), so anyone who can write the lockfile already has easier RCE — this is a defense-in-depth asymmetry, not a new primitive, hence Low.
- **Evidence** — `verified` (git behaviour, executed in first pass): mimicking `refspec_switch`, `git --git-dir=.git fetch --force --quiet --progress origin "--upload-pack=/tmp/gitinj/evilpack.sh"` against a local remote executed the script (`PWNED_via_upload_pack` created). `git checkout --force --quiet --upload-pack=…` only errors ("unknown option"), so the checkout sink is DoS/option-confusion, not code-exec. **`verified` (data flow, this pass — was `asserted`):**
  - *Within-file:* reading the whole file confirms `checkout/2` (line 123) computes `lock_rev = get_lock_rev(opts[:lock], opts)` at line 131 and consumes it at line 137 (`refspec_switch(opts, lock_rev || get_opts_rev(opts))`) and line 141 (`rev = lock_rev || …`) → line 142 sink. `get_lock_rev/2` (lines 303–309) returns the raw `lock` binary from the `{:git, repo, lock, lock_opts}` tuple with **no** `-`-prefix or shape check. The only refspec validation, `validate_refspec/1`, is reachable solely via `accepts_options` → `validate_git_options` (lines 41/53/57/235–239); it is never called on the fetch/checkout path. Confirmed: the two paths do not intersect.
  - *Out-of-file (neighbours read this pass):* `Mix.Dep.Lock.read/1` (`dep/lock.ex:25–36`) parses `mix.lock` with `Code.string_to_quoted` + `Code.eval_quoted` and returns the resulting map **verbatim** — it validates only for merge-conflict markers, never the rev value. `Mix.Dep.Converger.put_lock/2` (`dep/converger.ex:253–255`) then does `put_in(dep.opts[:lock], lock[app])`, attaching the raw tuple to `dep.opts[:lock]` — exactly what `get_lock_rev` reads — with no transformation. `Mix.Dep.Loader` likewise does not touch the rev. So no Mix layer between the lockfile and this file applies the `-`-prefix guard; the first-pass caveat is resolved, not merely assumed.
- **Suggested fix** — Apply the same `-`-prefix / type check to the locked revision before it is used as an argv element (e.g. validate in `get_lock_rev`, or prepend `--` where git accepts it — `git checkout --force --quiet -- <rev>` and pin the fetch refspec form), so a lock value can never be interpreted as a git option.

### Ruled out

- **Git URL `ext::`/`fd::`/`-`-prefixed injection via `opts[:git]`/`github:`** — the URL originates from a `mix.exs` dependency declaration, which Mix evaluates as code to resolve the dependency tree; an attacker who controls it already has arbitrary code execution by design (folder threat model: "installing evil code executes it is not a bug here"). Single-argv interpolation (`https://github.com/#{gh}.git`, `origin/#{branch}`) also cannot word-split into extra arguments under `System.cmd`. *(Dismissal basis: threat-model boundary + single-argv semantics; no untried input axis — the URL is not carried by a different transport than the code-eval that already trusts it.)*
- **`redact_uri/1` credential leak** — verified it correctly redacts the only realistic secret-bearing git URL form, `userinfo` (`https://user:token@host` → `****:****`, incl. `ssh://`). scp-style `git@host:path` carries no password and query-string tokens are not a standard git credential channel, so the gaps are not a meaningful secret disclosure; the credential is also the operator's own.
- **`subdir`/`sparse` `Path.join` `..` traversal (lines 154/163) and `File.write!(".git/info/sparse-checkout", sparse)` (line 176)** — values come from the trusted project/dependency `mix.exs` and only steer which subdirectory of the already-fetched repo is treated as the dep source; no path component crosses a boundary weaker than code-eval.
- **`parse_version`/`to_integer` crash on malformed version (lines 446–465)** — input is the output of the locally-installed `git --version`, a trusted local tool, not an attacker boundary.
- **`File.rm_rf!(opts[:checkout])` (line 107)** — the checkout path is computed by Mix (`deps/<app>`), not supplied through this file's API.

## Did I verify this?

- **`CSCAN-git-lock-rev-arg-injection` git behaviour** — verified by execution (first pass, unchanged): `git fetch … origin --upload-pack=<script>` against a local remote ran the script; `git checkout … --upload-pack=…` errors.
- **`CSCAN-git-lock-rev-arg-injection` data flow (both hedges)** — verified this pass by reading. Within-file: `get_lock_rev` (303) → 131 → 137/141 → 142 sink, disjoint from `validate_refspec` (only reachable via `accepts_options`). Out-of-file: `Mix.Dep.Lock.read` (`dep/lock.ex:25`) returns the eval'd map unvalidated; `Mix.Dep.Converger.put_lock` (`dep/converger.ex:253`) attaches it raw. No `-`-prefix guard exists on the path. Both prior hedges upgraded from `asserted` to `verified`.
- **`CSCAN-git-hookspath-literal-quotes`** — the *defect* (literal `''` directory instead of disabled hooks) is verified by execution; end-to-end RCE through Mix's exact 2.47.3 command sequence is **not** demonstrated and is stated as version/config-dependent. Confidence high on the broken control, not on live exploitation.
- **Ruled-out entries** — dismissed on trust-boundary grounds (inputs originate from code-eval'd `mix.exs`/`mix.lock` or a trusted local `git` binary) and on `System.cmd` single-argv semantics. These rest on the folder threat model, not on an untried input axis; no execution was required to exclude them.

## Risks / caveats

- I could not demonstrate end-to-end hook execution through Mix's precise command sequence on git 2.47.3, so `CSCAN-git-hookspath-literal-quotes` is a confirmed *broken control* whose exploitability is version/config-dependent (e.g. a future git that runs `post-checkout` during `submodule update`, or a different relative-path resolution, would turn the misdirected `''` into a live repo-controlled hook path). The defect itself — literal `''` instead of disabled hooks — is verified.
- Both `mix.exs` and `mix.lock` are evaluated as Elixir upstream of this file (`Mix.Dep.Lock.read/1` uses `Code.eval_quoted`), so the dominant code-execution risk in the git-dependency flow lives in the converger/lock reader, not here; that bounds the severity of everything reachable through this file's options, including `CSCAN-git-lock-rev-arg-injection`.
- I did not assess Windows-specific path/quoting behaviour (backslashes, `cmd.exe`), and `System.cmd` argument handling can differ there.

## Dropped in re-review

None. Both findings from the first pass are preserved; no dismissal was upgraded to a finding and no finding was downgraded. The only change is that the data-flow evidence for `CSCAN-git-lock-rev-arg-injection` moved from `asserted` to `verified` after reading `git.ex` in full and the two neighbour modules (`dep/lock.ex`, `dep/converger.ex`) that carry the lock value.
