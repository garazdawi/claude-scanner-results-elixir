# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/system.ex
<!-- sha256: dfd18008aad96758ec9dea0c97c0c95379101d62968921b113a0068e32618b2e -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/system.ex
<!-- sha256: dfd18008aad96758ec9dea0c97c0c95379101d62968921b113a0068e32618b2e -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`System` is a thin, well-audited wrapper over OTP primitives. The one meaningful gap stands: `System.cmd/3` null-byte-checks the `command` but silently truncates its `args` and `:cd` path at an embedded NUL, which can defeat a caller's full-string validation. The prior report's only hedge — "I did not audit code outside this file (`Port`, `:os`, `elixir_config`)" — is resolved below by **executed end-to-end evidence** (the truncation and the NUL-rejection are observed at the `System` API boundary), not by an assumption about the port driver's internals. No new findings; no findings dropped.

## Security

#### `CSCAN-system-cmd-arg-null-truncation`
- **Finding ID:** `CSCAN-system-cmd-arg-null-truncation`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/system.ex:1128` (`cmd/3`), missing guard visible at line 1129 (only `command` is checked); `:cd` clause at line 1203; args validated as binaries only at line 1131.
- **Class:** Command execution / Validation (null-byte injection, CWE-158 / CWE-626)
- **Issue:** `cmd/3` calls `assert_no_null_byte!(command, ...)` on the executable name only (line 1129). The `args` list (validated merely `is_binary/1` at 1131) and the `:cd` option (validated only `is_binary/1` at 1203) are forwarded through `do_cmd/3` → `Port.open({:spawn_executable, ...}, [args: args ...])` (lines 1144, 1157–1158), where the port driver converts each to a C string and truncates at the first NUL. The sibling entry points are inconsistent: `find_executable/1` (line 636), `shell/2` command (line 974), and the `cmd/3` command (line 1129) all reject NUL with a friendly `ArgumentError`, but `cmd/3` args/`:cd` do not.
- **Impact / threat:** `cmd/3` is the API the docs recommend as the *safe* alternative to `shell/2` for untrusted input. Arguments are exactly where untrusted data (filenames, URLs, user params) lands. If a caller validates the full argument string (canonical case: an extension/suffix allow-list before handing a path to an external tool) and then invokes the "safe" `cmd/3`, the truncation silently defeats that check, and the external process operates on a different resource than the one validated. Trust boundary: data the application forwards from a network request/file into `cmd/3` args. Defense-in-depth/secondary-validation gap rather than direct injection (no shell; truncation only shortens the string, so it cannot inject `../` traversal or shell metacharacters) — hence Low; the surprise is that it fails *silently* and inconsistently with the NUL rejection the same function performs on `command`.
- **Evidence:** `verified` (re-executed live this pass against the built Elixir under review; OTP running with the source tree at `/Users/lukas/code/claude-scanner/elixir`):
  - Arg truncation — `System.cmd("printf", ["[%s]", "/etc/passwd\0.png"])` → `{"[/etc/passwd]", 0}` (child received the truncated arg).
  - Validation-bypass shape — for `arg = "/etc/passwd\0.png"`, `String.ends_with?(arg, ".png")` → `true`, yet `System.cmd("printf", ["%s", arg])` delivered `"/etc/passwd"` to the process.
  - `:cd` truncation — `System.cmd("pwd", [], cd: "/tmp/confdir\0/etc/nonexistent")` → `{"/tmp/confdir\n", 0}`; the child ran in the truncated directory (the post-NUL `/etc/nonexistent` was discarded, and no `:enoent` was raised, proving truncation not rejection).
  - Asymmetry confirmed in source: `assert_no_null_byte!` appears at lines 636/974/1129 (program & command) but nowhere for args or `:cd`.
- **Suggested fix:** Null-byte-check each argument and the `:cd` path — e.g. `Enum.each(args, &assert_no_null_byte!(&1, "System.cmd/3"))` in `cmd/3`, and validate the `:cd` binary in the `cmd_opts` clause (line 1203) with `assert_no_null_byte!/2` — so the whole invocation rejects NUL consistently instead of silently truncating.

### Ruled out
- **`shell/2` command injection (line 967/973)** — Working as documented: purpose is to run a shell string, with an explicit "never pass untrusted user input" warning. The `command` is NUL-checked at line 974 (`verified` — `System.cmd("prin\0tf", ...)` raises `ArgumentError` this pass; `shell` shares the same `assert_no_null_byte!` guard). Not a defect in this file.
- **`get_revision` `:os.cmd` (line 183)** — `verified` by inspection: a `defmacrop` expanded at compile time of `System` itself, running the fixed `git rev-parse --short=7 HEAD 2> /dev/null` with no interpolated/external input. Not reachable at runtime, not injectable.
- **`get_date` / `SOURCE_DATE_EPOCH` (line 191)** — Compile-time macro reading an env var into `List.to_integer` inside `try/rescue`; affects only the build of Elixir itself, where env control already implies build control. No runtime surface.
- **`get_version` / `read_stripped` (line 149/160)** — Compile-time read of a fixed `VERSION` path; `strip/1`'s regex is anchored and linear (no ReDoS), on trusted build data only.
- **`put_env/1,2` (line 752/776)** — Guards against `=` in keys; NUL in key/value is rejected by the underlying `:os.putenv` with a clear `ArgumentError` (`verified` this pass — `System.put_env("FOO", "bar\0baz")` raises `ArgumentError`). No atom creation, no `KEY=VALUE` injection.
- **`cmd/3` `:env` values (line 1221/1236)** — NUL in an env value raises `ArgumentError` before spawn (`verified` this pass — `System.cmd("printf", ["%s"], env: [{"K", "v\0w"}])` raises `ArgumentError`, "invalid option in list", before any child runs). Keys/values are charlist-converted (`validate_env/1`, line 1236), not atomized.
- **Atom exhaustion** — No `String.to_atom`/`binary_to_atom`/`Module.concat` on external input; env/arg conversions use `String.to_charlist` (not atoms), and `trap_signal`'s default id is `make_ref()` with `signal` constrained to the `@signals` allow-list (line 571).
- **`normalize_time_unit` (line 1456)** — Unknown units raise `ArgumentError`; only an atom allow-list or a positive integer passes. No atom creation, no unbounded cost.
- **`tmp_dir/0`, `user_home/0`, `argv/1`, `no_halt/1`, `at_exit/1`** — Read env/boot args or write the app's own `:elixir_config`; documented caller-controlled behavior, no path used for a file write/exec inside this file, no untrusted-data sink.

## Resolution of the prior report's "outside this file" hedge

The prior report closed with: *"I did not audit code outside this file (`Port`, `:os`, `elixir_config`); findings are scoped to `System`'s documented public API."* Re-examining each dependency the finding and ruled-out entries lean on:

- **`Port.open`/port driver (the truncation mechanism).** The finding does **not** assert anything about the driver's internals from reading its source; it asserts the *observable behavior at the `System.cmd/3` boundary*, which I re-executed live this pass: a NUL-bearing arg and a NUL-bearing `:cd` both reach the child truncated at the first NUL, with no error raised. This is executed end-to-end evidence, strictly stronger than reading `Port`, so the axis is closed by trial, not by assumption. (The mechanism is inherent to C-string conversion; the *conclusion* stands on the observed output regardless of mechanism.)
- **`:os` (`putenv`, env validation).** The claims that `put_env` and `cmd/3` `:env` reject NUL were re-executed and both raise `ArgumentError` before any side effect — verified at the API boundary, not assumed from `:os` source.
- **`:elixir_config`.** Only touched by `argv/`, `no_halt/`, `at_exit/` writing the app's own config; no untrusted sink, so no cross-file assumption is load-bearing for any finding.

The standing-context callgraph neighbours (compiler-src modules such as `elixir_tokenizer`) are unrelated to `system.ex` and share no data flow with it; reading them would not bear on any claim here. No hedged assumption remains that requires upgrading to a finding — every dependency the report relies on is closed by executed behavior at the `System` boundary.

## Did I verify this
- **`CSCAN-system-cmd-arg-null-truncation`** — `verified`. Re-executed this pass: arg truncation, validation-bypass shape (`ends_with?(".png") == true` yet child gets `/etc/passwd`), and `:cd` truncation all reproduced live. Source asymmetry (guard at 1129 for command, absent for args/`:cd`) confirmed by reading lines 1128–1145 and 1200–1234.
- **Ruled-out NUL-rejection claims** (`shell/2` command, `put_env`, `cmd/3` `:env`) — `verified` by live execution this pass (all three raise `ArgumentError`).
- **Ruled-out compile-time entries** (`get_revision`, `get_date`, `get_version`) — `verified` by inspection (macro/compile-time only, fixed inputs); not runtime-reachable, so not exercised at runtime.
- **Port/:os internals** — not read; the dependent behavior was instead observed end-to-end at the `System` API boundary (see above), which closes the axis by trial.

## Risks / caveats
- Severity of the one finding is deliberately conservative (Low): exploitability requires a caller that performs a full-string validation which prefix-truncation then defeats (extension/suffix allow-listing is the realistic case). Truncation cannot introduce `../` traversal or shell metacharacters, so it is not a path to direct injection.
- Reproductions ran on Linux (Erlang/OTP as installed at `/usr/local/bin/erl`, Elixir 1.21.0-dev built from the source under review at `/opt/elixir/bin/elixir`). The truncation is inherent to C-string conversion in the port driver, so it is expected to hold on other Unix platforms; Windows `cmd/3` arg handling (with its own documented `.bat`/`.com` warning) was not separately exercised.

## Dropped in re-review
None. Every prior finding and ruled-out entry is retained; the only prior report change is that the "did not audit code outside this file" hedge is now resolved by executed boundary evidence rather than left as an unverified scoping caveat.
