# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/local.ex
<!-- sha256: 1a8f0d0d2d99359b29944355419a6c0e9e5c00bae7b8de60a2f720004a45603e -->

_Analysed on 2026-08-27T09:46:35Z_

## Summary

`Mix.Local` is safe apart from one Low-severity robustness gap: the CSV version-metadata parser reachable via `find_matching_versions!/3` throws uncaught exceptions on malformed rows instead of failing gracefully, so a compromised (or operator-CA-intercepted) Hex/Rebar metadata host can crash `mix local.hex` / `mix local.rebar` with a stacktrace.

## Security

#### `CSCAN-local-csv-metadata-parse-crash`
- **Finding ID** — `CSCAN-local-csv-metadata-parse-crash`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/mix/local.ex:211` (`find_version/4`), `:219` (`find_by_elixir_version/3` head), `:224` (`Version.compare/2`); input enters at `find_matching_versions!/3` (`:156`) → `read_path!/2` (`:164`) → `parse_csv/1` (`:193`).
- **Class** — Resource consumption (metadata parser crash / DoS)
- **Impact** — The signed-version CSV is fetched from `https://builds.hex.pm` (and the Rebar equivalent) and parsed here with no per-row validation. `parse_csv/1` produces ragged/empty rows verbatim, and three downstream sinks assume a well-formed 4+-column row: (1) `hd(&1)` in the `find_version/4` filter throws `ArgumentError` on an empty row (a line of only commas, e.g. `,,,`); (2) `find_by_elixir_version/3`'s single clause `[a, b, c, d | _]` throws `FunctionClauseError` on any row with fewer than 4 fields; (3) `Version.compare(hex_elixir_version, elixir_version)` throws `Version.InvalidVersionError` on a non-semver string in column 3. Any of these turns the intended graceful `Mix.raise("Could not find a version of #{name} matching: …")` into an unhandled crash of the install command. The metadata channel is TLS-`verify_peer` (passive MITM out of scope per the folder threat model), so the realistic triggers are a compromise of the metadata host or an operator-configured TLS-intercepting proxy (`HEX_CACERTS_PATH` / an `http://` `HEX_MIRROR`). Because a bad source already causes the install to fail, the security delta is only "ugly stacktrace vs. clean error" — hence Low.
- **Evidence** — `verified`. Installed `/opt/elixir/.../mix/local.ex` is byte-identical to the file under review (`diff -q` clean). Driving the public entry point with local CSV files:

```
$ printf ',,,\n'   > /tmp/csv1.csv   # comma-only row
$ elixir -e 'Mix.Local.find_matching_versions!("Hex","1.0.0","/tmp/csv1.csv")'
** (ArgumentError) … not a nonempty list
    :erlang.hd([])
    lib/mix/local.ex:211: anonymous fn/2 in Mix.Local.find_version/4

$ printf 'a,b\n'   > /tmp/csv2.csv   # short row, version=nil (the Rebar path)
$ elixir -e 'Mix.Local.find_matching_versions!("Hex",nil,"/tmp/csv2.csv")'
** (FunctionClauseError) no function clause matching in Mix.Local.find_by_elixir_version/3
    lib/mix/local.ex:219: Mix.Local.find_by_elixir_version/3

$ printf '1.0.0,deadbeef,notaversion,24\n' > /tmp/csv3.csv
$ elixir -e 'Mix.Local.find_matching_versions!("Hex",nil,"/tmp/csv3.csv")'
** (Version.InvalidVersionError) invalid version: "notaversion"
    lib/mix/local.ex:224: Mix.Local.find_by_elixir_version/3
```

- **Suggested fix** — Treat malformed rows as non-matching rather than fatal. In `find_by_elixir_version/3`, add a catch-all clause returning `nil` for rows that don't match the 4-field shape, and guard the version parse (`Version.parse(hex_elixir_version)` and skip on `:error`, or wrap `Version.compare` in a rescue). Replace the raw `hd(&1)` filter in `find_version/4` with a pattern that also skips empty/short rows (e.g. `&match?([^that_version | _], &1)`). All three then fall through to the intended `Mix.raise`.

### Ruled out
- **`name_for/2`, escript/archive name building** — inputs come from `mix.exs` project config (operator-controlled); not an attacker boundary.
- **`append_archives/0` → `Code.append_path` / `archives_tasks` / `archive_ebin` / `archives_ebins`** — paths derive from the operator's own installed-archive directory listing; installed archives already run as trusted plugins (per folder threat model), so loading them is not a new boundary crossing.
- **`check_elixir_version_in_ebin/1`** — reads `.elixir` and does `inspect(req)` / `Version.match?` on the content of an already-installed (operator-trusted) archive; a malformed requirement crashes but is self-inflicted, not attacker-reachable pre-trust.
- **`mix_paths/0` (`MIX_PATH`)** — operator environment; not remote/untrusted.
- **OTP-release string compare (`hex_otp_release <= otp_release`, `:225`)** — lexicographic compare is a latent correctness wart, but the CSV author already controls the entire returned tuple (version, digest, URL substitutions), so skewing eligibility grants no capability beyond what a compromised metadata source already has; no boundary crossed.
- **CSV fields flowing into the archive URL / `--sha512` in `local.hex.ex`** — that substitution lives in another file and is bounded by the same CSV-authenticity (TLS) trust root; not a defect in `local.ex`.

## Risks / caveats
- **Stale "verifies" docstring.** `find_matching_versions!/3` claims to fetch "signed CSV files, verifies," but there is no signature/`PublicKey` check anywhere in `lib/mix` anymore — the CSV's integrity rests solely on TLS (`verify_peer`), and the returned SHA-512 protects only the subsequently downloaded archive, not the CSV. This matches the folder threat model (metadata authenticity = TLS), so it is a documentation gap rather than an exploitable flaw, but it is worth correcting so callers don't assume cryptographic provenance of the metadata.
- **No size bound on the fetched CSV.** `parse_csv/1` itself is linear, but neither it nor `Mix.Utils.read_path`/`read_httpc` caps the response body, so a hostile metadata host could return a very large body and force large in-memory allocation. That cap belongs to `Mix.Utils` (out of this file's scope) and I did not pursue it here.
- **Reproduction fidelity.** The three crashes were driven through the real public function against a byte-identical build; I did not exercise them end-to-end over the network (`HEX_MIRROR`/proxy), relying instead on `Mix.Utils.read_path`'s documented local-file path, which shares the same `{:ok, binary}` → `parse_csv` flow.

