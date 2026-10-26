# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/runner.ex
<!-- sha256: 0037310e34eee06c14d0f186f9c6d7874c37dc1c08b8d86b8ab2fc759d526ee7 -->

# /Users/lukas/code/claude-scanner/elixir/lib/ex_unit/lib/ex_unit/runner.ex
<!-- sha256: 0037310e34eee06c14d0f186f9c6d7874c37dc1c08b8d86b8ab2fc759d526ee7 -->

_Re-reviewed on 2026-08-27T05:34:24Z (hedge-resolution pass)_

## Summary

`ExUnit.Runner` (`@moduledoc false`) is the internal driver that spawns, monitors, seeds, and reaps one process per test module, handles SIGQUIT flushing, and provisions per-test tmp dirs. Every input it touches is trusted developer test code, `@tag` metadata, or `mix test`/CLI config. In this pass I read the two neighbours the first pass hedged on — `ExUnit.Server` (`server.ex`, the module queue) and `ExUnit.Case` (`case.ex`, the `:tmp_dir` contract) — and I executed the four tmp-dir path axes the first pass only asserted. No sink is reachable across a network/disk/dependency/archive trust boundary. **No security defects found.**

## Security

No findings

### Ruled out

- **`create_tmp_dir!` → `Path.expand` + `File.rm_rf!`/`File.mkdir_p!` (lines 643–655).**\
  The final path is `["tmp", module, "#{name}-#{short_hash}", extra_path] |> Path.join() |> Path.expand()`, then recursively deleted and recreated. I executed every reachable component-shaping axis on OTP 29:
  - `Path.expand(["tmp","MyApp.Foo","testx-abcd1234","../../../../tmp/pwned"])` ⇒ `"/tmp/pwned"` — a traversal in `extra_path` **does** escape the tmp root.
  - `extra_path` is the documented `:tmp_dir` *string* tag. I confirmed in `case.ex:285–310` that this tag is set only by the developer (`@tag tmp_dir: "my_path"` / `@moduletag` / `@describetag`) and that "removed before being created" is the documented contract (`case.ex:289`). No network/disk/dependency input feeds it.
  - The `module` component is `escape_path(inspect(test.module))`; the `name` component is `"#{escape_path(to_string(test.name))}-#{short_hash}"`. I verified `escape_path("../evil") ⇒ "..-evil"` (the `/` maps to `-`, killing multi-segment traversal) and that the `name` slot always carries the `-#{short_hash}` suffix, so it renders as `..-abcd1234` and is **not** collapsed by `Path.expand` (ran: `/tmp/tmp/MyApp.Foo/..-abcd1234`). The `module` slot cannot be a bare `..` because `inspect/1` of any atom emits a `:`/`Elixir.` lead that `escape_path` rewrites to `-`. So the *only* traversal vector is `extra_path`, which is developer-owned.
  - This is self-inflicted by a test author who already has arbitrary code execution (threat model: the operator *is* the test author). Not a trust-boundary crossing.
  - **Correction to first pass:** the first pass claimed "no component can be exactly `..`" citing a name *prefix*. That reasoning is imprecise — I ran `escape_path(to_string(:"..")) ⇒ ".."`, i.e. a bare `..` *does* survive escaping. The name slot is safe not because of a prefix but because of the appended `-#{short_hash}` suffix (verified above). Verdict unchanged; rationale corrected.
  - **Axes examined:** `escape_path` on `../evil`, bare `..`, and module-`inspect` output; `Path.expand` collapse on a traversal `extra_path`; name-slot and module-slot `..` reachability; `extra_path` provenance (traced to `case.ex` `:tmp_dir` doc contract). **Not examined:** none of the in-file path-construction axes; the only residual is whether some *other* sibling could route non-developer input into the `:tmp_dir` tag — see the `run_module` provenance entry below, which I verified against `server.ex`.

- **`escape_path`/`@escape` incompleteness (lines 657–661).** The set escapes path separators and shell/glob metacharacters but not `.`. Ran: `escape_path("../evil") ⇒ "..-evil"`, `escape_path("..") ⇒ ".."`. Because `/` is escaped, no multi-segment traversal survives; the `.`-non-escaping only matters for a bare `..`, which cannot occupy the `module` slot (colon-escaped) and cannot occupy the `name` slot (suffix-appended). Input is developer-trusted regardless.
- **Axes examined:** escape output for `../`, `..`, and module-inspect strings; slot-by-slot `..` reachability. **Not examined:** none.

- **`run_module` / `apply(module, name, [context])` / `module.__ex_unit__(...)` dispatch (lines 241, 357, 526, 533).** Dynamic dispatch and code execution. I verified provenance in `server.ex`: the module queue is populated *only* by `ExUnit.Server.add_module/2` (`server.ex:16–37`), whose `name` is the compiled test-module atom and whose `parameterize`/`group` come from developer config; the `handle_call({:add, …})` clauses (`server.ex:140–173`) push these atoms directly onto `sync_modules`/`async_modules`/`groups` with no deserialization, no `binary_to_term`, and no external-input path. `take_async_modules`/`take_sync_modules` (`server.ex:43–49`, drained at `runner.ex:114,120`) return exactly those developer-origin entries. This is the framework's purpose, not an injection.
- **Axes examined:** every writer into the `server.ex` queue state (`:add`/`:restore_modules`), the read paths `runner.ex` consumes, and the shape of `name`/`group`/`params`. **Not examined:** none in `server.ex`; I did not walk every `case.ex`/`callbacks.ex` caller of `add_module`, but the queue's sole ingress is `add_module/2` and it carries only compiled-module atoms + developer config — no external decode sits between them.

- **`:erlang.term_to_binary(parameters)` (line 635) and `:erlang.md5` (line 638).** Serialize-only; there is no matching `binary_to_term` anywhere in the file (grep-confirmed). Ran the exact `short_hash` pipeline (`term_to_binary` → `md5` → `Base.encode16` → `binary_slice(0..7)`) to a lower-hex 8-char digest (`"df6a3534"`); it feeds only tmp-dir uniqueness, never a security decision.
- **Axes examined:** direction of serialization (encode-only), digest use-site. **Not examined:** none.

- **`:erlang.system_flag(:backtrace_depth, …)` (line 45), `Application.fetch_env!(:ex_unit, :after_suite)` callbacks (lines 70–71), `Process.put(@current_key, …)` (lines 320, 406).** VM-global / app-env / process-dict writes, all keyed on fixed internal atoms (`@current_key = __MODULE__`) or developer-supplied ExUnit config. No external key/value poisoning.
- **Axes examined:** key origin (fixed atom vs. config) for each write. **Not examined:** none.

- **SIGQUIT trap balance (lines 16–37).** `System.trap_signal(:sigquit, id, …)` is installed inside a `try` whose `after` always calls `System.untrap_signal(:sigquit, id)`, with `id = {__MODULE__, self()}`. The trap cannot leak into a later run.
- **Axes examined:** presence and unconditionality of the `after` untrap; `id` uniqueness. **Not examined:** none.

- **No eval/atom/deserialization/command/regex/base-decode sinks.** Grep confirmed the file contains none of `binary_to_term`, `String.to_atom`/`to_existing_atom`/`to_unsafe_atom`, `Code.eval`/`compile`, `:os.cmd`, `System.cmd`/`shell`, `Regex.compile`, or `Base.decode`.
- **Axes examined:** full-file grep for each sink class. **Not examined:** none.

## Risks / caveats

- **Trust model applied:** per the folder threat model, ExUnit executes trusted in-process test code with no untrusted peer; the test author already has arbitrary code execution. Sinks are judged by whether *external* (network/disk/dependency/archive/build) data can reach them; none can in this file, and I confirmed the module-queue ingress (`server.ex`) carries only developer-origin atoms + config.
- **Latent, not a finding:** `create_tmp_dir!` uses `Path.expand` + `File.rm_rf!` with no `Path.safe_relative`/root-confinement on `extra_path`, which I proved can escape `tmp/` (`⇒ /tmp/pwned`). It is safe today *only* because the `:tmp_dir` tag is developer-controlled (verified against `case.ex`). If any future caller ever routes untrusted input into that tag, this becomes an arbitrary-delete/zip-slip primitive; a defensive `Path.safe_relative` confinement would harden it. Flagged for awareness, not reported under the current API.
- **Cross-file sinks confirmed to live elsewhere:** the deserialization-shaped surfaces the folder model calls out — `FailuresManifest` unsafe `binary_to_term` and `ExUnit.Filters` `to_unsafe_atom` — are in sibling files, not `runner.ex`. This is a scoping statement, not an unverified assumption about `runner.ex`: grep confirms neither sink pattern appears in this file.

## Did I verify this?

- **Read the neighbours the first pass hedged on.** `ExUnit.Server` (`server.ex`, full file) — confirmed the module queue's sole writer is `add_module/2` with compiled-module atoms + developer config, no external decode. `ExUnit.Case` (`case.ex:270–318`) — confirmed the `:tmp_dir` tag is a documented developer-set string/boolean. This resolves the first-pass "single-file review" and "did not exhaustively audit those sibling modules for a smuggled-name/tag injector" hedges: the injector does not exist on the `server.ex` ingress path.
- **Executed the tmp-dir axes** (OTP 29 / Elixir 1.21.0-dev) rather than asserting them: `Path.expand` traversal ⇒ `/tmp/pwned`; `escape_path("../evil") ⇒ "..-evil"`; `escape_path("..") ⇒ ".."`; name-slot `..-hash` not collapsed; `short_hash` pipeline ⇒ `df6a3534`.
- **Did not re-audit** the genuinely out-of-file sinks (`FailuresManifest`, `Filters`) beyond confirming by grep they are absent from `runner.ex` — they belong to their own per-file reviews.

## Dropped in re-review

None. All first-pass ruled-out entries are preserved; one (`create_tmp_dir!`/`escape_path`) had its *rationale* corrected after execution (bare `..` survives escaping; the name slot is protected by the `-#{short_hash}` suffix, not by a prefix), with the verdict unchanged. No first-pass finding is dropped because the first pass reported none, and this pass likewise finds no defect — the developer-trust boundary holds and every previously-hedged assumption is now either executed (`Not examined: none`) or verified against the read neighbour.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 3 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **`apply(module, name, [context])` and `module.__ex_unit__(...)` dispatch (lines 241, 357, 526, 533).** Dynamic dispatch/code execution, but `module`/`name` originate from the `ExUnit.Server` queue populated by developer-compiled test modules — this is the framework's purpose, not an injection.

- **`create_tmp_dir!` → `Path.expand` + `File.rm_rf!`/`File.mkdir_p!` (lines 643–655).** The path is `["tmp", module, "#{name}-#{short_hash}", extra_path] |> Path.join() |> Path.expand()`, then recursively deleted and recreated. `extra_path` is unescaped and `Path.expand` collapses `..`, so a traversal string reaches a recursive delete — I verified `["tmp","MyApp.Foo","testx-…","../../../../tmp/pwned"] |> Path.join() |> Path.expand()` yields `/tmp/pwned`. But `extra_path` is the documented `:tmp_dir` string tag (`case.ex:294`, set only by the developer via `@tag tmp_dir: "..."` / `@moduletag`), and "remove before create" is documented behavior (`case.ex:289`). The module/name components are `inspect(test.module)` / `to_string(test.name)` — developer-authored, and `escape_path` rewrites `/`/`\`/`:` etc. to `-` so they cannot become a `..` component. This is self-inflicted by a test author who already has arbitrary code execution; no external (network/disk/dependency) input reaches the tag. Not a trust-boundary crossing.

- **No eval/atom/deserialization/command/regex/base sinks.** Grep confirmed the file contains none of `binary_to_term`, `String.to_atom`/`to_existing_atom`, `Code.eval`/`compile`, `:os.cmd`, `System.cmd`/`shell`, `Regex.compile`, or `Base.decode`.
