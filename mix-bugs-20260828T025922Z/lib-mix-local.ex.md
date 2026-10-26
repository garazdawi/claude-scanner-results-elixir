# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/local.ex
<!-- sha256: 1a8f0d0d2d99359b29944355419a6c0e9e5c00bae7b8de60a2f720004a45603e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:17:57Z_

## Summary

`Mix.Local` is an internal (`@moduledoc false`) helper module for archives/escripts and for fetching Rebar/Hex install metadata; the naming/doctest logic is sound, but the CSV-metadata parser used by `find_matching_versions!/3` assumes every downloaded row is well-formed and crashes with an uncaught exception on malformed input.

## Bugfixes

#### `CSCAN-local-csv-row-non-total`
- **Finding ID:** `CSCAN-local-csv-row-non-total`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `elixir/lib/mix/lib/mix/local.ex:219-228` (`find_by_elixir_version/3`), reached from `find_matching_versions!/3` (line 156).
- **Class:** Missing clause / non-total function
- **Impact:** `find_matching_versions!/3` is documented to fetch signed CSV metadata (used to install Rebar and Hex from S3) and return the matching version tuple, raising a clean `Mix.raise` ("Could not find a version … matching") only when nothing matches. But its inner parser is non-total: any CSV line that does not split into at least four comma fields hits no function clause and raises `FunctionClauseError`, and any line whose third field is not a valid version string raises `Version.InvalidVersionError`. A truncated download, a trailing partial line, or any format drift in the network-supplied CSV therefore aborts the install with an opaque internal stack trace that tells the caller nothing about what to fix, instead of the intended "could not find a matching version" message.
- **Evidence:** `verified`.
  - Malformed short row (`elixir /tmp/t.exs`), CSV `1.0.0,abcdef,1.14.0,24\n badrow\n`: output `RAISED: FunctionClauseError: no function clause matching in Mix.Local.find_by_elixir_version/3`.
  - Short row on the artifact-filter path, CSV `1.0.0,abcdef\n` with version `"1.0.0"`: `RAISED2: FunctionClauseError`.
  - Non-version elixir column, CSV `1.0.0,abcdef,not_a_version,24\n`: `RAISED3: Version.InvalidVersionError`.
- **Suggested fix:** Add a catch-all clause to `find_by_elixir_version/3` (e.g. `def find_by_elixir_version(_, _, _), do: nil`) so unrecognized rows are skipped rather than crashing, and/or guard the `Version.compare` call so an invalid version string in a row is treated as non-matching. That preserves the documented "no match → `Mix.raise`" contract for genuinely malformed metadata.

## Risks / caveats

- The module is `@moduledoc false`; I treated its `@doc`'d/exported functions (`name_for/2`, `find_matching_versions!/3`, `archive_*`, etc.) as the reviewable public surface per the task's "documented API" rule. Whether a truncated/malformed CSV is realistically reachable depends on upstream signature verification that is **not** performed in this file (the `@doc` claims the CSV is "verified", but no verification occurs here) — so the crash's reachability rests on how much the callers trust/validate the downloaded bytes before this function; the parser itself is unconditionally non-total.
- Both doctests for `name_for/2` were run and pass.
- I did **not** exhaustively test the code-path/archive functions (`append_archives`, `check_elixir_version_in_ebin`, etc.), which depend on filesystem/`Code.append_path` state; they were reasoned about only.

### Ruled out
- **`hex_otp_release <= otp_release` string comparison (line 225)** — comparing OTP release numbers as strings is lexicographic, which would misorder e.g. `"9"` vs `"10"`. Ruled out as a *current* bug: `System.otp_release()` and the Hex builds CSV have used ≥2-digit, equal-width release numbers for many years, so realistic values compare correctly; it is only a latent issue if single-digit or 3-digit ("100") releases ever appear. Not reachable with any input a well-behaved caller produces today.
- **`name_for/2` with missing `:app`** — produces `"-0.1.0.ez"` / `".ez"`; this is a caller passing an incomplete project config (contract violation), not a defect in the function.
- **CRLF (`\r`) in CSV last field** — `parse_csv` splits only on `\n`, leaving a trailing `\r` on the `hex_otp_release` field; verified harmless for the `<=` comparison with current values and does not reach the digest/version fields, so no wrong result.

