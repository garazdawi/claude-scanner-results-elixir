# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/version.ex
<!-- sha256: 8336eb75fc9875a07d345f4ab112ee076f94e0bb8dc437c1822f88e4756299da -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:00:13Z_

I've verified the module's behavior extensively against its documentation and SemVer semantics using the Elixir build (1.21.0-dev) compiled from this source. Everything checks out.

## Summary

`version.ex` is a faithful, well-tested SemVer parser/matcher; extensive behavioral probing of every documented public entry point (`match?/3`, `compare/2`, `parse/1`, `parse!/1`, `parse_requirement/1`, `parse_requirement!/1`, `to_string/1`) found no reachable defect.

## Bugfixes

No findings.

### Ruled out

- **`compare/2` `~>` upper-bound pre-release exclusion (`match_op?/3`, lines 192–202)** — I suspected the `[0]` sentinel pre on the upper bound might mis-order real pre-releases of the upper bound. Verified `~> 2.0`, `~> 2.1-dev`, and `~> 2.1.2` all correctly exclude `3.0.0-rc`/upper-bound pre-releases regardless of `:allow_pre`, matching the moduledoc table (lines 72–104). Basis: verified (ran all table rows, outputs matched).
- **Pre-release precedence in `do_compare`/`Requirement.compare` (lines 220–234, 379–393)** — I suspected Erlang term ordering (`pre1 > pre2`) could diverge from SemVer for mixed numeric/alphanumeric and unequal-length identifier lists. Verified `alpha < alpha.1`, `alpha.1 < alpha.beta`, numeric `1 < alpha`, and release `> pre` — all SemVer-correct because numeric identifiers become integers (integer < binary in term order = numeric < alphanumeric). Basis: verified.
- **`allow_pre: false` semantics for `>`, `>=`, `<`, `<=` (lines 204–218)** — Confirmed `<`/`<=` ignore `:allow_pre` and `>`/`>=`/`~>` honor it per the doc table (rows for `>= 2.1.0` and `< 2.2.0`). Basis: verified.
- **Lexer/`revert_lexed` `and`/`or` handling and implicit `==` (lines 554–621)** — I checked for precedence errors, mis-inserted implicit `==`, and malformed-input crashes. `and` binds tighter than `or`; `"== == 2.0.1"`, `"1.0.0 2.0.0"`, and empty/omitted-patch operands all return `:error` rather than crashing; implicit `== ` works. Basis: verified.
- **Numeric-component / leading-zero / digit-limit enforcement (lines 641–699)** — Verified leading-zero rejection (major and numeric pre), 14-digit acceptance vs 15-digit rejection, empty pre/build rejection, alphanumeric pre with leading zeros allowed, and build leading-zeros allowed. All match the moduledoc (lines 21, 29). Basis: verified.
- **`to_string/1` round-trip and nil-build handling (lines 520–536)** — `"1.14.0-rc.0+build0"` round-trips; nil build yields no `+` suffix. Basis: verified.
- **Doc wording "Patch segments are compared lexicographically if they are alphanumeric" (line 336)** — This sentence sits in `compare/2`'s doc and actually describes *pre-release* identifier comparison; patch is always numeric (parser rejects non-digit patch), so the alphanumeric case is unreachable for patch. It is a pre-existing upstream wording imprecision that mislabels the segment name, not a behavioral defect — the described comparison behavior genuinely exists for pre-release parts. Too weak and non-behavioral to report as a bug. Basis: verified (parser rejects non-numeric patch; pre-release ordering confirmed above).

## Risks / caveats

- The `Version.Requirement` internal `match?/2`, `compare/2`, and lexer/parser paths are `@doc false` but reachable only through the public API, which always feeds them well-formed lexed structures; I treated hand-crafted malformed `Requirement` structs as out-of-scope internal misuse.
- The sandbox Elixir (`1.21.0-dev`) is built from this source, so all "verified" claims reflect this exact file. I exercised the documented entry points and doctest examples directly but did not run the full upstream ExUnit suite.

