# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/local.rebar.ex
<!-- sha256: 20f210408d230b34a60b80df1438bbd725daa73518936792184ad8d7fe7abb18 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/local.rebar.ex
<!-- sha256: 20f210408d230b34a60b80df1438bbd725daa73518936792184ad8d7fe7abb18 -->

_Re-analysed on 2026-08-27_

Re-review of the first pass. Every hedge (`read_httpc` TLS branches "inspected enough / out of scope"; the CSV-crash "belongs to Mix.Local"; checksum-before-write "confirmed by reading the delegated code") has been discharged by reading the three callgraph neighbours — `Mix.Utils.read_path`/`read_httpc`/`checksum` (`utils.ex`), `Mix.Local.find_matching_versions!`/`find_by_elixir_version` (`local.ex`), `Mix.Rebar.local_rebar_path` (`rebar.ex`) — and by executing the `String.replace`/`Version.compare`/short-row axes in an Elixir shell. No hedge required upgrading to a finding; each verified in the safe (downgrade) direction.

## Summary
`Mix.Tasks.Local.Rebar` is thin glue that routes both the operator path (`mix local.rebar rebar3 <path>`) and the no-arg S3 path through the TLS-verified `Mix.Utils.read_path`, which performs checksum verification *before* returning the bytes that `install_from_path` writes to a fixed, System-derived local path. No security issue is reachable through its documented public API.

## Security

No findings

### Ruled out

- **URL construction from CSV values (`String.replace`, lines 120–124).**\
  `rebar_version` (= CSV `artifact_version`) and `otp_release` (= CSV `hex_otp_release`) are network-derived and are *not* version-validated, and cross-placeholder expansion does occur. But the base authority is always `hex_url`, and the placeholders sit strictly in the URL *path* (`hex_url <> "/installs/[ELIXIR_VERSION]/rebar3-[REBAR_VERSION]-otp-[OTP_RELEASE]"`), after the fixed `://builds.hex.pm/installs/` prefix, so a CSV value can never reach the authority component. `elixir_version` (= CSV `hex_elixir_version`) is additionally constrained: `Mix.Local.find_by_elixir_version` (local.ex:224) calls `Version.compare(hex_elixir_version, elixir_version)` and only returns the row if it parses, and `Version` strings admit neither `/` nor `[`. A malicious CSV therefore still requires a compromised trust anchor (Hex CDN or `HEX_BUILDS_URL` mirror), which already grants full escript+checksum control → RCE by documented design.\
  **Confidence: High.**\
  Axes examined: literal-vs-backreference replacement; cross-placeholder value expansion; authority-injection via injected `@`/`//`; `elixir_version` validation path.\
  Not examined: none.\
  Executed evidence (Elixir shell): `String.replace("a[X]b", "[X]", "\0\1")` → `"a\0\1b"` (literal, no `\N` backreference); injecting `"3.20[OTP_RELEASE]"` as `rebar_version` → `".../rebar3-3.2027-otp-27"` (cross-expansion stays in path); injecting `"@evil.com/"` → `"https://builds.hex.pm/installs/@evil.com//x"` (the `@` lands in the path, authority stays `builds.hex.pm`); `Version.compare("[REBAR_VERSION]", "1.18.0")` raises `Version.InvalidVersionError` (placeholder text cannot survive the elixir-version slot).

- **TLS verification on the remote-fetch path and its retry branches (delegated to `Mix.Utils.read_httpc`, utils.ex:862–926).**\
  Read in full. Base `ssl_options` (utils.ex:888–892) are `verify: :verify_peer` + `customize_hostname_check` via `:public_key.pkix_verify_hostname_match_fun(:https)`. Both retry branches preserve these: the IP-family fallback (utils.ex:908–912) re-issues with the *same* `http_options`; the TLS-alert branch (utils.ex:914–917) sets `middlebox_comp_mode: false` — a TLS-1.3 middlebox-compatibility toggle, not a verification setting — leaving `verify`/`customize_hostname_check` untouched. `HEX_CACERTS_PATH` swaps only the CA source (`:cacertfile`), which is an operator-set env var. No verification downgrade exists on any branch.\
  **Confidence: High.** Axes examined: verify-mode across both retry branches; hostname-check retention; CA override provenance. Not examined: none.

- **Checksum sourced from the same CSV as the payload.**\
  The `sha512` used for verification (`Keyword.put(opts, :sha512, sha512)`, line 126) comes from the same TLS endpoint as the escript, so it gives no protection against a compromised source — this is the intentional S3-install design (identical for Hex and Rebar). TLS provides authenticity; the checksum guards integrity/corruption. Not a defect of this file. **Confidence: High.**

- **Checksum-before-write ordering (delegated to `Mix.Utils.read_path`, utils.ex:803–828).**\
  Read in full. On the URL branch, `read_path` runs `with {:ok, binary} <- read_httpc(path), do: checksum(binary, opts)` inside the async task; `checksum/2` (utils.ex:832–848) returns `{:checksum, message}` on mismatch and `{:ok, binary}` only when the digest matches (or no `--sha512` was supplied). `install_from_path` writes (line 85) exclusively in the `{:ok, binary}` arm, so a mismatched payload never reaches `File.write!`. Checksum-before-use holds. **Confidence: High.** Axes examined: ordering of fetch→checksum→return; which arm reaches `File.write!`. Not examined: none.

- **Missing checksum on the operator path.**\
  `mix local.rebar rebar3 <url>` without `--sha512` installs without an integrity check, but the operator chose the URL and TLS authenticates it. Operator-trust, out of scope. **Confidence: High.**

- **Executable file write (`File.write!` + `File.chmod!(_, 0o755)`, lines 85–86).**\
  Installs an executable escript, but the target is `Mix.Rebar.local_rebar_path(:rebar3)` (rebar.ex:59–68), which is `Path.join([Mix.Utils.mix_home(), "elixir", "#{major}-#{minor}-otp-#{System.otp_release()}", "rebar3"])` — derived entirely from `System.version()` and `System.otp_release()`, with no attacker-controlled component. No traversal/TOCTOU on the target. **Confidence: High.** Axes examined: external influence on every path segment. Not examined: none.

- **Non-constant-time checksum compare.**\
  `checksum/2` (utils.ex:832) compares with `!=` (`actual when actual != expected`), but both operands are public (hash of public content vs. public expected digest); no secret is compared, and the code lives in `Mix.Utils`, not this file. **Confidence: High.**

- **Error-handling clauses (`:badpath`, `{:local,_}`, `{kind,_} when kind in [:remote,:checksum]`, lines 89–107).**\
  All are reachable via the corresponding `read_path` return values (`:badpath` utils.ex:826; `{:local, _}` utils.ex:858; `{:remote, _}` utils.ex:817/941/944; `{:checksum, _}` utils.ex:843). None are dead defensive code. **Confidence: High.**

## Risks / caveats (cross-file — reachable via this file's API but not attributable to it)

- **Malformed CSV crashes `Mix.Local`, fail-closed (verified).** On the no-arg S3 path, a CSV row with fewer than four columns has no matching clause in `Mix.Local.find_by_elixir_version/3` (local.ex:219, head `[artifact_version, digest, hex_elixir_version, hex_otp_release | _]`, no catch-all) → uncaught `FunctionClauseError`; a well-formed-arity row whose elixir-version field is non-parseable crashes `Version.compare/2` with `Version.InvalidVersionError` (local.ex:224). Both are fail-closed aborts (nothing is written/installed) inside `lib/mix/local.ex`, and both require the trusted Hex/mirror endpoint to serve malformed metadata. Belongs to `Mix.Local`, not this file. Executed evidence: a 2-element row pushed through the `[a,b,c,d|_]` head raises `FunctionClauseError`; `Version.compare("[REBAR_VERSION]", "1.18.0")` raises `Version.InvalidVersionError`. **Confidence: High (that the crash is real and fail-closed); the routing decision that it belongs to `Mix.Local` is a scoping call, not a defect claim.**

## Did I verify this?
- `read_httpc` TLS options and both retry branches — **read in full** (utils.ex:862–926); confirmed no verify/hostname downgrade. (First-pass hedge "out of scope / inspected enough" — now discharged.)
- `read_path` fetch→checksum→return ordering and `checksum/2` — **read in full** (utils.ex:803–848); confirmed checksum-before-write. (First-pass hedge "confirmed by reading the delegated code" — now cited to specific lines.)
- `Mix.Rebar.local_rebar_path/1` — **read in full** (rebar.ex:59–68); confirmed fixed, System-derived target.
- `Mix.Local.find_matching_versions!`/`find_by_elixir_version` — **read in full** (local.ex:156–228); confirmed the CSV crash shape and that `elixir_version` is version-validated before URL use. (First-pass hedge "belongs to Mix.Local" — now verified concretely.)
- `String.replace` literal/cross-placeholder/host-injection and `Version.compare` behaviours, plus the short-row `FunctionClauseError` — **executed** in an Elixir shell (outputs quoted above).
- A live end-to-end `mix local.rebar` network download — **not run** (no network in the sandbox); this affects none of the security conclusions above, all of which were established by reading the delegated code and running the isolated behaviours.

## Dropped in re-review
None. Every ruled-out entry and caveat from the first pass is preserved; the only change is that each first-pass hedge is now backed by a full read of the named neighbour and/or executed evidence, and the confidence/axes fields are made explicit.
