# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.uninstall.ex
<!-- sha256: bc99bd844f3934a22ff7c446de35293e965053a39a213082471958bbffc6a6e1 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.uninstall.ex
<!-- sha256: bc99bd844f3934a22ff7c446de35293e965053a39a213082471958bbffc6a6e1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28_

## Summary

`archive.uninstall.ex` is a 28-line `Mix.Task` wrapper whose `run/1` contains no logic of its own and delegates entirely to `Mix.Local.Installer.uninstall/4`. I re-read the two callgraph neighbours the first pass hedged on (`Mix.path_for/1` in `mix.ex` and `Mix.Local.Installer.uninstall/4` in `local/installer.ex`) and confirmed both the existence and the exact signatures the call relies on. No in-scope defects.

## Bugfixes

No findings.

The file's entire behavior, now re-verified against the neighbours:

- `@impl true def run(argv)` — correct arity-1 `run` satisfying the `Mix.Task` behaviour (TASK_RUN_ARITY_ONE). Confirmed in file at line 24–27.
- It calls `Mix.path_for(:archives)`. **Verified** at `mix.ex:655–659`: `path_for(kind)` is declared and the `:archives` head returns `System.get_env("MIX_ARCHIVES") || Path.join(Mix.Utils.mix_home(), "archives")` — a `Path.t()`, exactly the first positional argument `uninstall/4` expects.
- It calls `Mix.Local.Installer.uninstall/4` with `(Mix.path_for(:archives), "archive", argv, @switches)`. **Verified** at `installer.ex:300–301`: `@spec uninstall(Path.t(), String.t(), OptionParser.argv(), keyword) :: Path.t() | nil` and `def uninstall(root, listing, argv, switches)`. The four arguments (`root` path, `listing` string `"archive"`, `argv`, `switches` keyword) match positionally and by type.
- `@switches [force: :boolean]` matches the single documented `--force` option (file line 16–17, 20–22).

There is no untrusted-input parsing, resource acquisition, state threading, pattern-matching, or arithmetic in this file — no Phase 1 sink class has an applicable site here.

## Risks / caveats

- The substantive logic — `OptionParser.parse!` (line 302), the `File.exists?`/`matching_package` lookup (306–309), the `cond` that returns `nil` when no package is found or the user declines, and `File.rm_rf!` (312–314) — lives in `Mix.Local.Installer.uninstall/4`. This is a genuinely separate reviewed file, not a hedge: `archive.uninstall.ex` neither wraps nor conditions any of that behavior, so a defect there is attributable to `local/installer.ex`'s own review and cannot be exercised differently through this wrapper. This is a scoping fact, not an unverified assumption.
- No repro was executed because the file has no independently exercisable behavior: every observable effect is produced by `Mix.path_for/1` and `Mix.Local.Installer.uninstall/4`, both of whose signatures I read and confirmed this pass.

## Ruled out

- **Signature mismatch on `uninstall/4` (arity / argument order / types)** — Confidence: high. Axes examined: arity, argument order, static type of each positional arg (path / string / argv / keyword) against the `@spec` and function head at `installer.ex:300–301`. Not examined: none — the call site is a static 4-tuple of literals/one call, fully determined by reading both ends; there is no runtime-varying input that could make arity or positional shape disagree.
- **`Mix.path_for(:archives)` unsupported / wrong return type** — Confidence: high. Axes examined: clause existence for the `:archives` atom and return type (`Path.t()`), verified at `mix.ex:655–659`. Not examined: none — `:archives` is a compile-time literal in the call, so no other argument value can reach this clause from this file.
- **Wrong `@switches` shape passed to option parsing** — Confidence: high. Axes examined: `@switches` is a literal `[force: :boolean]` keyword passed as the `switches` argument, which `uninstall/4` feeds directly to `OptionParser.parse!(argv, switches: switches)` (installer.ex:302); keyword-list shape matches. Not examined: none — the value is a compile-time literal with no variance.

## Did I verify this?

Yes for every claim in this report:
- Read `archive.uninstall.ex` in full (28 lines).
- Read `mix.ex:655–663` — confirmed `path_for(:archives)` exists and returns a path.
- Read `installer.ex:295–314` — confirmed `uninstall/4`'s `@spec`, head, and argument usage.

The three ruled-out entries carry `Not examined: none`: every axis is a compile-time-fixed property of this file's call site (literal atom `:archives`, literal string `"archive"`, literal keyword `@switches`, and `argv` forwarded unchanged), so there is no runtime-varying input axis left to push through both implementations. Reading the neighbours was sufficient here precisely because no argument this file supplies is data-dependent.

## Dropped in re-review

None. The first pass reported no findings; all three implicit "out of scope for this file" hedges were resolved by reading the two neighbour modules and are now stated as verified scoping facts (delegated logic genuinely belongs to `local/installer.ex`) rather than unverified assumptions. No hedge required upgrading to a low-confidence finding, because the wrapper introduces no input, guard, or state of its own that could carry a defect.
