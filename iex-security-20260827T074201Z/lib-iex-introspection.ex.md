# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/introspection.ex
<!-- sha256: 35f85d16c0b95928301fe539f9d4e3d70eed35dbf7f6fc018a4ec3ffdc501f1b -->

# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/introspection.ex
<!-- sha256: 35f85d16c0b95928301fe539f9d4e3d70eed35dbf7f6fc018a4ec3ffdc501f1b -->

_Re-analysed on 2026-08-27_

## Summary

`IEx.Introspection.open_location/2` builds a shell command from a filesystem path escaped only with `inspect/1` (Elixir string-quoting, not shell-quoting) and runs it through `:os.cmd/1`, so a hostile source-file name — reachable via `open(module)` on any module whose compiled-in `:source` path an attacker influenced — yields OS command execution (High, unchanged).

Re-review of the two first-pass hedges ("out of scope for this file", "did not exhaustively") turned both into executed evidence. The terminal-escape passthrough and the quadratic typespec-formatting cost are now filed as low-confidence findings rather than declined caveats, and the `String.to_unsafe_atom` dismissal is upgraded because the "implausibly gigantic module" bound does not hold against a *forged* BEAM (the same attacker model the High finding already assumes).

## Security

#### `CSCAN-introspection-editor-os-cmd-injection`
- **Finding ID** — `CSCAN-introspection-editor-os-cmd-injection`
- **Severity** — High
- **Confidence** — high
- **Location** — `elixir/lib/iex/lib/iex/introspection.ex:172` (and the template branch at :164–170), function `open_location/2`; reached from `open/1` (:106–149) and `source_location/1` → `source_mfa/3` → `rewrite_source/2`.
- **Class** — Command execution (shell) / Template-interpolation (CWE-78)
- **Issue** — `open_location/2` constructs the editor command by string interpolation and escapes the file path with `inspect(file)`:

```elixir
if editor =~ "__FILE__" or editor =~ "__LINE__" do
  editor |> String.replace("__FILE__", inspect(file)) |> String.replace("__LINE__", Integer.to_string(line))
else
  "#{editor} #{inspect(file)}:#{line}"
end
{:ok, :os.cmd(String.to_charlist(command))}
```

`inspect/1` produces Elixir double-quoted-string escaping (it escapes `"` and `\`), **not** shell escaping. It leaves backticks and `$()` untouched, and `:os.cmd/1` runs the string through `/bin/sh -c`, where command substitution fires *inside* double quotes. Any shell metacharacter in `file` therefore executes. The path is not caller-typed data only: `open(module)` derives `file` from `module.module_info(:compile)[:source]` via `rewrite_source/2`, whose `_` branch returns the raw compiled-in source path (`List.to_string(source)`) verbatim for any module that is not part of Elixir/OTP. The `File.regular?(file)` gate is satisfied whenever a real file with that name exists on disk.

- **Impact** — A dependency, archive, or git checkout that ships a source file whose *name* contains shell metacharacters (e.g. `` evil`cmd`.ex `` or `evil$(cmd).ex` — legal Unix filenames), or a hand-built BEAM with a forged `:source` attribute plus a matching on-disk file, turns the routine developer action `open(SomeModule)` / `open({file, line})` into arbitrary OS command execution on the developer's machine. It requires only `EDITOR`/`ELIXIR_EDITOR` to be set (near-universal) and does not require the target module to execute any of its own code — a wholly benign module with a hostile source path is enough. This is an "inspect untrusted code → RCE" escalation.

- **Evidence** — `verified`.
- `inspect/1` leaves metacharacters unescaped: `inspect("pwn`touch A`.ex")` → `"pwn`touch A`.ex"`; `inspect("pwn$(touch B).ex")` → `"pwn$(touch B).ex"`.
- End-to-end via `open_location/2` (EDITOR=`echo`), file named `` pwn`touch MARKERPWNED`.ex ``: `File.regular?(fname) => true`; `open_location returned => {:ok, ~c"pwn.ex:1\n"}`; `/tmp/MARKERPWNED` created.
- End-to-end via `open(module)` through the compiled `:source` path — source file `` evil`touch MARKER2`.ex `` compiled to `EvilMod`, then `IEx.Introspection.open(EvilMod)`: compile source attr `~c"/tmp/evil`touch MARKER2`.ex"`; `/tmp/MARKER2` created.
- The `__FILE__`/`__LINE__` template branch is equally injectable (`MARKER3` created with `ELIXIR_EDITOR="echo E __FILE__ at __LINE__"`).

- **Suggested fix** — Do not hand off a shell string. Split the editor command into program + args and invoke without a shell, e.g. `System.cmd(program, args ++ ["#{file}:#{line}"], into: IO.stream())`, so the path is a single argv element that no shell parses. If a shell invocation must be retained for editor strings that themselves contain shell syntax, POSIX-shell-quote the path (wrap in single quotes and escape embedded single quotes) instead of using `inspect/1`.

---

#### `CSCAN-introspection-terminal-escape-passthrough` — *Upgraded from hedged assumption in first pass*
- **Finding ID** — `CSCAN-introspection-terminal-escape-passthrough`
- **Severity** — Low
- **Confidence** — low
- **Location** — `print_source/2` (:228, `IO.puts`); `b/1` definition printer (:641, `IO.puts(definition)`); `t/1` type printer (:776, `IO.puts`); `print_typespec/1` (:897, `IO.puts(types)`); indirectly `format_typespec/3` output (:864–871) and `print_doc/5` headings.
- **Class** — Output neutralization / terminal-escape injection (CWE-150 / CWE-116)
- **Issue** — The in-file printing sinks emit strings drawn from a possibly-untrusted BEAM (source paths, callback/type definitions rendered from the module's typespec chunk, doc headings) straight to the terminal with no stripping of ANSI/C0 control bytes. First pass declined this as "the heavy rendering happens in `IO.ANSI.Docs`, out of scope"; on re-review the *in-file* `IO.puts` sinks are themselves the injection point and were tested directly:
  - `Path.relative_to_cwd("lib/foo\e[31mHACK\e[0m\a.ex")` → `"lib/foo\e[31mHACK\e[0m\a.ex"` — ESC and BEL survive into `print_source/2`'s output.
  - A type whose *name* atom contains control bytes formats to `"@type my\e[31mtype\a() :: t()"` through the exact `Code.quoted_to_algebra |> Inspect.Algebra.format` pipeline `format_typespec/3` uses — `contains ESC? true`.
- **Impact** — A hostile module (the same "inspecting untrusted code" model as the High finding) can inject terminal escape sequences into the developer's session via `source/1`, `b/1`, `t/1`, `h/1`. Consequences are the usual terminal-injection class: cursor manipulation, screen clobbering, misleading output, and on vulnerable/permissive terminal emulators clipboard or title-bar abuse. Low severity because it requires the developer to actively introspect an attacker-supplied module and the payload is limited to what a terminal will act on.
- **Evidence** — `verified` (control-byte passthrough reproduced for both the path sink and the typespec-name sink on Elixir 1.21.0-dev / OTP 29; outputs quoted above).
- **Suggested fix** — This is arguably better fixed once at the IO layer, but the file-local mitigation is to route these `IO.puts`/`IO.write` calls through a control-character scrubber (strip C0 except `\t`/`\n`, and CSI/OSC sequences) before printing untrusted-BEAM-derived strings.

---

#### `CSCAN-introspection-typespec-algebra-quadratic-dos` — *Upgraded from hedged assumption in first pass*
- **Finding ID** — `CSCAN-introspection-typespec-algebra-quadratic-dos`
- **Severity** — Low
- **Confidence** — low
- **Location** — `format_typespec/3` (:864–871): `Code.quoted_to_algebra() |> Inspect.Algebra.format(IEx.Config.width())`; reached from `get_spec/3` (`h/1`), `format_type/1` (`t/1`), and `format_callback/1` (`b/1`).
- **Class** — Uncontrolled resource consumption / algorithmic complexity (CWE-407 / CWE-834)
- **Issue** — First pass hedged: "I did not exhaustively audit every `format_typespec` → `Code.quoted_to_algebra` path for cost blowups." Re-review tried it. A typespec AST with deeply nested constructors (`list(list(... t ...))`) fed through the exact pipeline scales super-linearly:

  | nesting depth | wall time |
  |---|---|
  | 1000 | 9.6 ms |
  | 2000 | 34.0 ms |
  | 4000 | 131.5 ms |
  | 8000 | 564.0 ms |

  Each doubling of depth ≈ quadruples the time — O(depth²), consistent with the known `Inspect.Algebra.format` re-descent cost (`fits?` re-walking groups). The AST here comes from `Code.Typespec.spec_to_quoted` / `type_to_quoted` on the module's abstract-type chunk, which a crafted BEAM controls; unlike the general `Inspect` case there is no `limit` cap capping depth (the document is built in full by `Code.quoted_to_algebra` regardless of width), so the default path is exposed.
- **Impact** — A forged/hand-built BEAM carrying a pathologically nested spec or type makes `h(Mod.fun)`, `t(Mod)`, or `b(Mod)` burn seconds-to-minutes of CPU per introspection call, hanging the developer's IEx session. Self-inflicted (developer must introspect the hostile module) and single-session, hence Low.
- **Evidence** — `verified` (timings above, reproduced on Elixir 1.21.0-dev / OTP 29 through `Code.quoted_to_algebra |> Inspect.Algebra.format(80) |> IO.iodata_to_binary`).
- **Suggested fix** — This is fundamentally the `Inspect.Algebra.format` quadratic tracked in [[inspect-algebra-review]]; file-local mitigation is to bound typespec nesting depth (or the rendered size) before formatting.

### Ruled out
- **`Macro.expand/2` (:21, :30, :55) in `decompose/2`** — the AST and `context` originate from what the user typed at the IEx prompt (compile/eval of the caller's own expression); a user who can supply that AST already has full code execution. No external-data crossing. **Axes examined:** data provenance (prompt-local) | **Not examined:** none.
- **`Code.ensure_loaded/1`, `module.module_info/1`, `mod.behaviour_info/1`** — these dispatch on the caller-named module and run compiler-generated (pure) functions; for a hand-forged BEAM the compromise already occurred at load time (`@on_load`), which is inherent to loading untrusted modules, not a defect of this file. **Axes examined:** dispatch target trust, load-time-vs-introspection-time | **Not examined:** none.
- **`:beam_lib.chunks/2` (:283)** — parses an on-disk BEAM but every failure shape is absorbed by the `with … else _ -> {source, nil, nil}` fallback; a malformed chunk degrades to "source not available," not a crash or memory sink here. **Axes examined:** malformed/absent chunk (falls through `else`) | **Not examined:** in-BEAM abstract-code size amplification (a giant valid `raw_abstract_v1` chunk drives the `Enum.reduce` over `code` at :286, but that is linear in a chunk the attacker must physically ship; no super-linear factor found).

### Upgraded (was ruled out in first pass)
- **`String.to_unsafe_atom/1` (`translate_callback_name_arity`, :726)** — moved from "Ruled out" into a low-confidence concern. First pass dismissed it as "cannot approach atom-table exhaustion without an implausibly gigantic module." That bound assumes a *legitimately compiled* module. The attacker model for this file (already assumed by the High finding) is a **forged BEAM**, whose `callbacks` chunk can enumerate arbitrarily many distinct `MACRO-<name>` entries. `get_callback_docs/2` maps `translate_callback/1` over *every* entry from `Typespec.fetch_callbacks/1` **before** any filtering, and `translate_callback_name_arity/1` calls `String.to_unsafe_atom(macro_name)` unconditionally — `to_unsafe_atom` bypasses the existing-atom check, so each distinct name mints a new atom. `b/1`, `h/1`, and `t/1`-adjacent paths all funnel through this. This is CWE-770 unbounded atom creation reachable by introspecting a hostile module.
  - **Severity/Confidence:** Low / low. Low confidence because I did **not** build the forged-BEAM PoC (the callbacks-chunk forgery is fiddly and I did not run it end-to-end); the reachability argument is read from the call chain, not executed. Kept low severity because it is self-inflicted (developer must run `b`/`h` on the hostile module) and single-node. **Axes examined:** call-order (conversion precedes filter), `to_unsafe_atom` semantics (creates unconditionally) | **Not examined:** end-to-end forged-BEAM atom-count growth (not run).

## Risks / caveats
- The command-injection finding was reproduced on the sandbox toolchain (Elixir 1.21.0-dev / OTP 29) on Linux, where `:os.cmd/1` uses `/bin/sh -c`. On Windows the shell differs but the missing-escaping defect is the same class.

## Did I verify this?
- **`open_location` OS command injection** — **Verified, executed.** `inspect/1` non-escaping confirmed; both the default branch and the `__FILE__`/`__LINE__` branch fire injected commands; reached end-to-end through `open(module)` via a forged compiled `:source` path (markers created on disk).
- **Terminal-escape passthrough** — **Verified, executed** (upgraded from a first-pass "out of scope" hedge). `Path.relative_to_cwd/1` and the `Code.quoted_to_algebra`/`Inspect.Algebra.format` typespec pipeline both preserve ESC/BEL bytes into the in-file `IO.puts` sinks; outputs quoted in the finding. Filed as Low/low.
- **Typespec-formatting quadratic DoS** — **Verified, executed** (upgraded from a first-pass "did not exhaustively" hedge). Measured O(depth²) growth (9.6 → 34 → 131 → 564 ms doubling depth) through the exact `format_typespec/3` pipeline. Filed as Low/low.
- **`String.to_unsafe_atom` atom creation** — **Reachability read from the call chain, NOT executed.** The forged-BEAM atom-growth PoC was not run; left as Low/low with `Not examined: end-to-end forged-BEAM atom-count growth`, per the rule that an untried axis keeps the entry at low confidence rather than a clean dismissal.
- **`Macro.expand`, module-info dispatch, `:beam_lib.chunks` fallback** — Ruled out by provenance/trust reasoning and the `else` fallthrough; `Not examined: none` except the abstract-code chunk-size axis noted inline (linear, attacker must ship the bytes).

### Dropped in re-review
None. Every finding and caveat from the first pass is preserved; the two declined caveats and the one dismissal were upgraded (not dropped) with executed or call-chain evidence.
