# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/local.hex.ex
<!-- sha256: d4af8ab22774b0b267041553c67820d5e835f637e7da9f84bc09abcd4528e414 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:58:14Z_

## Summary

`local.hex.ex` is the `mix local.hex` task that installs Hex; it is small, and the one substantive defect is a flipped version comparison in the `--if-missing` reinstall decision that inverts the intended "install when the present Hex is too old" semantics.

## Bugfixes

#### `CSCAN-local.hex-if-missing-version-compare-flipped`
- **Finding ID:** `CSCAN-local.hex-if-missing-version-compare-flipped`
- **Severity:** Medium
- **Confidence:** medium
- **Location:** `lib/mix/lib/mix/tasks/local.hex.ex:54`, `run/1` (the `if_missing` + `version` branch)
- **Class:** Operator / precedence (comparison direction)
- **Impact:** For `mix local.hex VERSION --if-missing` (the form used in automation/bootstrap to guarantee *at least* a given Hex), the direction of `Version.compare/2` is reversed. When the already-installed Hex is **older** than the requested `VERSION`, the comparison yields `:lt`, the condition is false, and the outdated Hex is **not** upgraded — the user asked for `VERSION` and silently keeps the old one, later hitting failures from a too-old Hex. Conversely, when the installed Hex is **newer** than `VERSION`, the comparison yields `:gt`, the condition is true, and Mix reinstalls/**downgrades** an already-present newer Hex — the opposite of what `--if-missing` implies. The correct test is `== :lt` (install when the installed version is below the requested one).
- **Evidence:** verified.
  - `Version.compare/2` direction:
    ```
    $ elixir -e 'IO.inspect Version.compare("1.0.0","2.0.0"); IO.inspect Version.compare("2.0.0","1.0.0")'
    :lt
    :gt
    ```
    so `Version.compare(installed, requested)` is `:lt` when the installed version is older.
  - Effect on the decision, simulating an installed Hex `2.0.0` with a requested `2.0.6`:
    ```
    installed=2.0.0 requested=2.0.6
    current code (:gt) should_install? = false   (outdated Hex NOT upgraded)
    intended (:lt)     should_install? = true
    ```
    The current `== :gt` code declines to upgrade an outdated Hex, which is the reverse of the branch's evident intent.
- **Suggested fix:** Change the comparison on line 54 from `== :gt` to `== :lt`:
  ```elixir
  not Code.ensure_loaded?(Hex) or Version.compare(Hex.version(), version) == :lt
  ```

## Risks / caveats

- I could not diff against an authoritative upstream copy (the tree under review is not a git checkout and contains no test for this task), so the "intended" direction rests on semantic reasoning about `--if-missing VERSION` (ensure *at least* `VERSION`) plus the confirmed `Version.compare/2` behavior, not on a recovered original. This is why confidence is `medium` rather than `high`.
- Secondary, not reported as a finding: in the same branch, if a caller passes a non-full version string (e.g. `"2.0"`), `Version.compare/2` raises `Version.InvalidVersionError` rather than a task-friendly message — but a partial/invalid version violates the documented `[version]` contract, so it is out of scope.
- The network/`Archive.Install` path (`run_install/2`) was not executed (no Hex/network in the sandbox); the review of that path is by inspection only, and I found nothing wrong there — the URL template substitution and `--sha512`/`--force` argv threading are consistent with `local.rebar.ex`.

