# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io/ansi.ex
<!-- sha256: dd04b22d5da0c40be0aa6ebee374a0da696c263dc4ffc0465b9ad71e8122ca18 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:14:23Z_

I have full coverage. Every public function is traced, every sink checked, and the interesting behaviors are verified by running against the built-from-source Elixir. Writing the report.

---

## Summary

`IO.ANSI` is a pure ANSI-escape-sequence producer/chardata formatter with no code-execution, filesystem, network, deserialization, runtime-atom, or dynamic-dispatch sinks reachable through its documented public API; no security findings.

## Security

`No findings`

The module's entire attack surface is: two config reads (`enabled?/0`, `syntax_colors/0`), integer-guarded escape builders (`color/1,3`, `color_background/1,3`, `cursor/2`, `cursor_up|down|left|right/1`), the static `defsequence`-generated accessors, and the `format/2` / `format_fragment/2` chardata walkers. All were traced to their inputs and exercised; none crosses a boundary into a dangerous sink.

### Ruled out

- **Deep-nesting / recursion DoS in `format/2` (`do_format/5`, lines 315–342).** Every non-terminal clause ends in a tail call to `do_format`, using the `rem` argument as an explicit heap-allocated work stack rather than the call stack. Basis: **verified** — `IO.ANSI.format([<500,000-deep nested list>, :red, "x"], true)` completed and returned correct iodata (`iodata_size=10`) with no stack overflow. Cost is linear in input size; no super-linear blowup.

- **ANSI/control-byte injection via `format/2` binary parts.** `do_format` passes binary and char (non-list, non-atom) terms through unchanged (line 328–330). Basis: **verified** — `IO.ANSI.format(["prefix", "\e[2J\e[1;1H malicious"], true)` yields `"prefix\e[2J\e[1;1H malicious"`. This is by contract: the module is documented as *rendering* ANSI and `format` only converts **atoms** to codes; it never claims to sanitize. An attacker-controlled binary already contains whatever bytes it contains before reaching `format`, which neither adds nor amplifies escape sequences. Sanitizing untrusted terminal output is an application/`IO`-layer concern, not this file's contract, so this is not a defect here. (See Risks.)

- **Escape-code smuggling via the atom API.** `format_sequence/1` matches a fixed set of literal atoms and raises `ArgumentError` on anything else (line 264–266); it performs no atom creation, `apply`, or dispatch. Basis: **verified** — `:reset` → `"\e[0m"`; `:totally_unknown_code` raises `"invalid ANSI sequence specification: :totally_unknown_code"` under both `emit?: true` and `emit?: false` (validation runs even when codes are not emitted). Arbitrary escape sequences cannot be injected through the atom channel.

- **Atom-table exhaustion (CWE-770).** The only atom creation is `String.to_unsafe_atom/1` at lines 164, 198, 201, 204, all inside module-top-level comprehensions over the static literals `[1..9]` and the fixed `colors` list — evaluated at **compile time** on developer-controlled data, unreachable from any runtime input. `format_sequence/1` creates no atoms. Basis: verified by reading (compile-time position, literal inputs) plus grep confirming no other atom sink.

- **Interpolation/format-string injection in escape builders.** `color/1,3`, `color_background/1,3`, and `cursor/*` interpolate values into `"\e[…"` strings, but each is guarded to a bounded integer (`0..255`, `0..5`, `is_integer and >= 0`, `>= 1`). Integers cannot carry escape metacharacters. Basis: **verified** — `color(999)`, `cursor(-1,0)`, and `color(:atom)` all raise `FunctionClauseError` (a caller contract violation, explicitly out of scope), never producing a malformed/injected sequence.

- **Config-poisoning via `enabled?/0` / `syntax_colors/0` (lines 53–55, 81–83).** Both only *read* `:elixir` application env; writing that env requires code execution already, so it is not a trust boundary. Worst case from a poisoned value is cosmetic (emitting/suppressing color codes). No external-input write to shared state exists in this file.

- **Dead defensive code.** The `do_format` clause order was checked: clause 1 (`[term|rest]`) matches only non-empty lists, so atom clauses 2/3 are reachable; the `is_boolean(emit?)` entry guard makes clauses 2/3 (true/false) exhaustive for atoms; clause 4 (`not is_list`) handles binaries/chars; the `format_sequence(other)` catch-all (line 264) is reachable via any unknown atom (verified above). No guard is unreachable.

## Risks / caveats

- **Not a sanitizer — by design.** The single most likely real-world misuse is an application passing untrusted data through `IO.ANSI.format/2` expecting control characters to be neutralized. They are not: binary/char parts pass through verbatim (verified). This is correct for the module's documented contract, but downstream code that writes formatter output containing untrusted bytes to a real terminal remains responsible for stripping dangerous control sequences. That risk lives at the call site, not in this file, so it is not a finding here.
- **`String.to_unsafe_atom/1`** is used only at compile time on static literals; I did not audit that function's own implementation (out of scope — different file), but no runtime or external-input path reaches it from this module.
- Coverage basis: I read the entire 343-line file and enumerated every `def`/`defp` head; behavioral claims were exercised against `/opt/elixir/bin/elixir`. Assertions about compile-time evaluation and clause reachability are from reading the source structure rather than execution, but are corroborated by the runtime tests above.

