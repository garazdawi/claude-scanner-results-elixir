# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/archive.check.ex
<!-- sha256: 592235f18f0493e7808761572163d4c6fd6f61ca9d959b698491e30e32aac587 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:35:53Z_

## Summary

`archive.check.ex` is a small `Mix.Task` (`run/1`) that validates the `:archives` project option; I found no reachable behavioural defect — the one suspicious spot (interpolating a parsed requirement struct into the mismatch error) renders correctly because `Version.Requirement` implements `String.Chars`.

## Bugfixes

No findings

## Risks / caveats

- I verified (`elixir -e`) that `Version.Requirement` implements `String.Chars`, so the `#{req}` interpolation at line 37 produces a clean message (`~> 2.0.0`) rather than a `Protocol.UndefinedError`. Likewise `#{vsn}` on a charlist and `List.to_string(vsn)` both behave as intended.
- I did not exercise the full `Mix.Project.config()` / `Application.load` path against a real installed archive; I reasoned about those from their documented return shapes (`Application.spec/2` → `nil` when absent, feeding the `is_nil(vsn)` branch).
- One edge left as a caveat, not raised as a finding: if an installed archive's `.app` `vsn` is not valid SemVer (e.g. `"1.0"`), `Version.match?/2` at line 35 raises `Version.InvalidVersionError` rather than a `Mix.raise`. This requires a malformed archive rather than valid documented input, and the raised message is itself informative, so it does not meet the bar for an in-scope bug.

### Ruled out

- **`#{req}` at line 37 raising a protocol error** — `req` is the compiled `Version.Requirement` struct, but I confirmed by running that it implements `String.Chars` and interpolates to `~> 2.0.0`. No crash.
- **`#{vsn}` / `List.to_string(vsn)` on a charlist** — `vsn` from `Application.spec/2` is a charlist; both interpolation and `List.to_string` handle it correctly (verified: `foo-1.0.0`).
- **`parse_archive/1` totality** — the two heads cover the documented `{atom, binary}` shape and an `other` catch-all that `Mix.raise`s with a clear message; no unhandled shape falls through.
- **Ignored `Application.load/1` return (`_ = ...`)** — an already-loaded or missing app is handled downstream by the `is_nil(vsn)` check, which raises the correct "could not be found" message; discarding the return is intentional and safe.
- **`Version.parse_requirement` error path** — line 52 correctly matches `:error` and raises a descriptive message using the still-binary `req`.

