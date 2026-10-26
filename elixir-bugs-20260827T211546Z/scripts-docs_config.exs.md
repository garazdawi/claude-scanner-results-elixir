# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/docs_config.exs
<!-- sha256: dd0d033db30da9b96100e9929cb584345e4a70518af62719fb1cceea03c37c0b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:05:24Z_

## Summary

The script correctly generates `docs_config.js` on well-formed input, but two unguarded operations over git-tag data (`Version.parse!/1` and `Enum.fetch!/2`) abort the entire docs-config generation on tag configurations the code otherwise anticipates.

## Bugfixes

#### `CSCAN-docs_config-tag-parse-crash`
- **Finding ID** — `CSCAN-docs_config-tag-parse-crash`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `docs_config.exs:21` (the `version = Version.parse!(rest)` binding inside the `versions` comprehension)
- **Issue** — The comprehension keeps every tag matching `"v" <> rest` that does not end in `-latest`, then unconditionally calls `Version.parse!(rest)`. `Version.parse!/1` raises `Version.InvalidVersionError` on any `v`-prefixed tag that is not a full `major.minor.patch` semver string. Because this is a plain match (`=`), not a `<-` generator, it cannot filter — it raises and unwinds the whole script.
- **Class** — Missing clause / non-total function (unguarded `parse!` on partially-trusted input).
- **Impact** — A single stray tag such as `vnext`, `v1.16`, or any non-semver `v…` tag aborts `docs_config.exs` before `File.mkdir_p!`/`File.write!` run, so no `docs_config.js` is produced and the docs build fails with an error naming only the version fragment, not the offending tag. The presence of the explicit `-latest` filter shows the author expects non-plain-version `v` tags to exist, yet the parse is unguarded, so the safety depends entirely on tag-naming discipline.
- **Evidence** — `verified`. In a synthetic repo (tags `v1.14.0`, `v1.15.0`, `v1.16.0`, plus `vnext`): `elixir lib/elixir/scripts/docs_config.exs elixir` →
  ```
  ** (Version.InvalidVersionError) invalid version: "next"
      (elixir 1.21.0-dev) lib/version.ex:438: Version.parse!/1
      lib/elixir/scripts/docs_config.exs:21: anonymous fn/2 ...
      lib/elixir/scripts/docs_config.exs:19: (file)
  ```
- **Suggested fix** — Replace the unconditional `parse!` binding with a filtering match, e.g. add a generator over `Version.parse(rest)` keeping only `{:ok, version}` (and continue filtering `Version.compare(version, skipped) == :gt`), so unparseable `v` tags are skipped instead of crashing.

#### `CSCAN-docs_config-no-stable-fetch-crash`
- **Finding ID** — `CSCAN-docs_config-no-stable-fetch-crash`
- **Severity** — Low
- **Confidence** — high
- **Location** — `docs_config.exs:32-37` (`latest = versions |> Stream.filter(&(&1.pre == [])) |> Enum.fetch!(0) |> Version.to_string()`)
- **Issue** — `latest` takes the first version with an empty pre-release list. If every version greater than `1.0.3` is a pre-release (empty stable set), `Enum.fetch!(0)` raises `Enum.OutOfBoundsError`.
- **Class** — Missing clause / non-total function (`fetch!` with no empty-list handling).
- **Impact** — In a repo state where only pre-release tags exist above the `skipped` floor, generation aborts with an opaque `Enum.OutOfBoundsError` instead of falling back (e.g. to `System.version()`). Not reachable for the real Elixir repo, which always has stable tags `> 1.0.3`, so impact is confined to unusual/early tag states.
- **Evidence** — `verified`. Synthetic repo with only `v1.15.0-rc.0` and `v1.16.0-rc.1`: `elixir lib/elixir/scripts/docs_config.exs elixir` →
  ```
  ** (Enum.OutOfBoundsError) out of bounds error at position 0 when traversing enumerable #Stream<...>
      (elixir 1.21.0-dev) lib/enum.ex:1080: Enum.fetch!/2
      lib/elixir/scripts/docs_config.exs:36: (file)
  ```
- **Suggested fix** — Use `Enum.fetch/2` (or `Enum.find/2`) and fall back to `System.version()` (or the highest available version) when no stable version exists.

## Risks / caveats
- Both findings are verified crashes but require tag configurations that the current Elixir repository does not exhibit (all `v` tags are valid semver or end in `-latest`, and stable releases `> 1.0.3` always exist); they are latent robustness gaps rather than failures on today's expected input. The `-latest` filter's existence is what makes the parse-crash finding non-hypothetical — the tag namespace already contains non-plain-version `v` tags.

### Ruled out
- **`app` shadowing at line 59 (`for app <- ~w(...)s`)** — the comprehension variable does not leak, so line 63's `File.mkdir_p!("doc/#{app}")` and line 48's URL still use the `System.argv()` value. Verified: full run produced correct `doc/elixir/...` path and `elixir.hexdocs.pm` URLs, and the compiler emits no shadowing warning (empty stderr). Basis: verified.
- **`latest` marking logic (lines 32-56)** — highest stable version is correctly selected from the descending-sorted list and exactly one node gets `"latest":true`. Verified against tags `v1.14.0`/`v1.15.0`/`v1.16.0-rc.0`/`v1.16.0`: `v1.16.0` marked latest, `1.0.2`/`1.0.3` skipped, `-latest` tag skipped. Basis: verified.
- **Non-git branch (lines 26-40)** — produces `versionNodes = []` and `searchNodes` at `System.version()`, with the documented `IO.warn`. Verified in a non-git dir. Basis: verified.
- **`JSON.encode_to_iodata!/1` availability and map/atom-key encoding** — function exists in the built Elixir (1.21.0-dev) and encodes the atom-keyed maps correctly. Basis: verified.

