# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/hex.ex
<!-- sha256: 431728cc068f7b38d250c9dcaca5c0393bf152876331a21a99b2b4f3c8b0ece3 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/hex.ex
<!-- sha256: 431728cc068f7b38d250c9dcaca5c0393bf152876331a21a99b2b4f3c8b0ece3 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Mix.Hex` is a small internal helper (four public functions plus one private) that checks/installs/updates Hex and resolves the builds URL. In this re-review I eliminated both "outside this file" hedges from the first pass by (a) reading the full delegate chain `Mix.Tasks.Local.Hex.run/1` → `Mix.Tasks.Archive.Install.run/1` → `Mix.Local.Installer.install/3` → `local_install/3`, and (b) executing `Version.match?/2` against a spread of pre-release inputs on the sandbox toolchain (Erlang/OTP 29). The install path returns a boolean literally, and the pre-release characterization in my first-pass caveat was **inaccurate and is corrected below**. No reachable behavioural defect.

## Bugfixes

No findings.

Function-by-function inventory against the Phase 1 classes (return-shape / error-propagation / resource classes):

- **`ensure_installed?/1`** — `cond` returns `true`, `Mix.Tasks.Local.Hex.run(["--force"])`, or `false`. The delegate return type is now fully traced (not assumed) — see Ruled out #1. `@spec … :: boolean` and the `@doc` hold.
- **`ensure_updated?/0`** — three `cond` branches (`not loaded` → `false`; version mismatch → prompt, install-and-`exit({:shutdown,0})` or fall through to `false`; else → `true`). The `if` with no `else` correctly falls to the trailing `false`. Matches the documented contract.
- **`start/0`** — `catch kind, reason` covers `:error`/`:throw`/`:exit`, prints guidance, re-raises with `:erlang.raise/3` preserving `__STACKTRACE__`. Symmetric; no swallow.
- **`url/0`** — `||` env fallback chain (`HEX_BUILDS_URL` → `HEX_MIRROR` → default). No crash path.

## Ruled out

- **1. `ensure_installed?/1` install branch could return a non-boolean (DOCUMENTED_RETURN_SHAPE)** — Confidence: high. **Verified, not assumed.** Read the whole chain: `Mix.Tasks.Local.Hex.run(["--force"])` (`local.hex.ex:47-70`) computes `should_install? = true` (no `--if-missing`), returns `run_install(nil, ["--force"])`; `run_install/2` (`:72-89`) returns `Mix.Tasks.Archive.Install.run([url, "--sha512", sha512, "--force"])`; that (`archive.install.ex:90-92`) returns `Mix.Local.Installer.install/3`; for a `{:url, _}` spec (an `http(s)` URL parsed by `parse_args/2`, `installer.ex:210-216`) `install/3` reaches `local_install/3`, which returns the literal `true` (`installer.ex:174`) or `false` (`:176`). No branch of this path yields a non-boolean. Axes examined: return-value on every branch of the delegate chain (read directly). Not examined: none — return shape is a static literal, fully determined by reading.

- **2. `Version.match?(Hex.version(), ">= 2.0.6")` spuriously rejects legitimate pre-release Hex versions in `ensure_updated?/0` (ERROR_PROPAGATES / behavioural)** — Confidence: high. **Corrects an inaccurate first-pass caveat.** Executed `Version.match?/2` on the sandbox (`elixir vtest.exs`, OTP 29). Results: `2.0.6`→true, `2.1.0`→true, `2.1.0-rc.0`→**true**, `3.0.0-dev`→**true**, `2.0.6-rc.1`→false, `1.9.0`→false. The default `Version.match?/2` uses `allow_pre: true`, so future pre-releases (e.g. `2.1.0-rc.0`) *do* satisfy `>= 2.0.6`. The only pre-release excluded is one of the exact lower boundary (`2.0.6-rc.1`), which genuinely sorts below `2.0.6` and is correctly `< 2.0.6`. My first-pass statement that pre-releases are excluded "unless the requirement itself carries a pre-release" was wrong; there is no problematic exclusion. Axes examined: newer-minor pre-release, dev pre-release, boundary pre-release, exact match, below-boundary release (all pushed through `Version.match?/2` and compared to expected SemVer ordering). Not examined: none.

- **3. `url/0` treats empty-string env var as set (behavioural quirk)** — Confidence: high. Executed previously and unchanged: with `HEX_BUILDS_URL=""`, `System.get_env/1` returns `""`, which is truthy, so `url/0` returns `""` rather than falling through. This is the ubiquitous Elixir `||`-env idiom, triggered only by an operator explicitly exporting an empty value, and matches ecosystem convention. Intended convention, not a defect. Axes examined: unset, empty-string, non-empty (the three env states through the `||` chain). Not examined: none.

- **4. `start/0` swallows Hex startup failure (silent-swallow class)** — Confidence: high. The `catch` re-raises via `:erlang.raise(kind, reason, __STACKTRACE__)` after printing guidance; the error propagates with original kind/reason/stacktrace. No swallow. Axes examined: `:error`/`:throw`/`:exit` kinds (all covered by the two-arg `catch` clause, re-raised). Not examined: none.

## Did I verify this?

- **Delegate return-shape (hedge #1, "outside this file"):** Yes — read `local.hex.ex`, `archive.install.ex`, and `installer.ex` end-to-end. The boolean return is now a read fact, no longer a static assumption about an unread file. **Upgraded from hedged assumption in first pass to verified.**
- **`Version.match?/2` pre-release semantics (hedge #2, "outside this file"):** Yes — ran it against six inputs on OTP 29 and compared to expected SemVer ordering. The first-pass caveat was factually wrong and is corrected; no defect exists. **Upgraded from hedged assumption to verified (and corrected).**
- **`url/0` empty-string / env chain:** Yes — executed previously, all three env states.
- **`start/0` re-raise:** Yes — by inspection of the `catch`/`:erlang.raise` pair in this file.
- **Not exercised end-to-end:** the live network install against `builds.hex.pm` (requires network). This does not affect any conclusion — every return-shape claim rests on read literals, not on runtime execution of the fetch.

## Dropped in re-review

- **Dropped: first-pass caveat "Version.match?/2 excludes pre-release Hex versions from `>= 2.0.6` unless the requirement itself carries a pre-release."** New evidence: executed `Version.match?/2` (Ruled out #2) shows the default is `allow_pre: true` — newer pre-releases match, and only the below-boundary `2.0.6-rc.1` is (correctly) rejected. The caveat's premise was inaccurate; its "not a bug" conclusion stands more strongly. Replaced by the corrected Ruled out #2 entry rather than silently removed.
