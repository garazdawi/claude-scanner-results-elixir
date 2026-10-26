# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/local.rebar.ex
<!-- sha256: 20f210408d230b34a60b80df1438bbd725daa73518936792184ad8d7fe7abb18 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:59:16Z_

## Summary

`Mix.Tasks.Local.Rebar` is a small, well-formed Mix task that installs `rebar3` either from a local path/URL or from Hex's CDN; its argument dispatch, option handling, checksum/tuple threading into `install_from_path`, and error-case coverage all line up with the helper contracts they depend on, and I found no genuine reachable defect.

## Bugfixes

No findings

## Risks / caveats

- Review was confined to `local.rebar.ex`. I read the collaborators it depends on (`Mix.Local.find_matching_versions!` / `find_by_elixir_version`, `Mix.Utils.read_path`) to confirm the return-shape and tuple-order contracts this file relies on, but did not audit those files in depth for their own bugs — they are out of scope.
- Behavioural claims about network/S3 install (`install_from_s3`) were reasoned from the helper contracts, not executed end-to-end (no CDN access in the sandbox). The tuple destructuring `{elixir_version, rebar_version, sha512, otp_release}` was verified against `find_by_elixir_version/3`'s literal return `{hex_elixir_version, artifact_version, digest, hex_otp_release}` (local.ex:226), which matches positionally.

### Ruled out

- **`skip_install?/2` — `and` vs `|>` precedence** (line 129–134): I suspected the pipe chain might not be grouped as the right operand of `and`. Verified by running the exact structure: `false and …` short-circuits to `false`, and `true and File.exists?(<absent>)` yields `false` — parses and behaves as intended (`a and ((manager |> path) |> exists?)`). Correct.
- **`install_from_path/3` non-total `case` on `read_path` result** (lines 82–107): `Mix.Utils.read_path/2` returns exactly `{:ok, binary}`, `:badpath`, `{:local, msg}`, `{:remote, msg}`, `{:checksum, msg}` (utils.ex:803–828 + `checksum/2`). Every shape is matched (`:remote`/`:checksum` share a guarded clause). Total.
- **S3 tuple order vs URL replacement** (lines 117–124): destructured order `{elixir, rebar, sha512, otp}` matches the producer's return order, and each is substituted into the matching `[..]` placeholder. Consistent.
- **`--sha512` override in S3 path** (line 126): `Keyword.put(opts, :sha512, sha512)` intentionally replaces any user-supplied checksum with the CSV digest for the resolved artifact — expected behaviour, not a bug.
- **`run/1` arg dispatch** (lines 51–63): `["rebar3", path | _]` requires ≥2 args with a literal `"rebar3"` head; all other shapes (bare `rebar3`, missing keyword, `rebar2`) fall through to the explicit `Mix.raise`. No missing clause; `OptionParser.parse` (3-tuple, invalid-tolerant) is a deliberate choice, and `if_missing`/`sha512`/`force` map correctly to `@switches`.
- **`not skip_install?/…` boolean-operator use** (lines 67, 73): `if_missing` is a `:boolean` switch, so `skip_install?` always returns a genuine boolean, making `not` safe.

