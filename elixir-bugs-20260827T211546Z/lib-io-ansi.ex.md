# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io/ansi.ex
<!-- sha256: dd04b22d5da0c40be0aa6ebee374a0da696c263dc4ffc0465b9ad71e8122ca18 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:20:33Z_

I've reviewed the file thoroughly and verified behavior against the built Elixir toolchain. Here is my report.

## Summary
`IO.ANSI` is a small, well-worn escape-sequence module whose executable paths (`format/2`, `format_fragment/2`, color/cursor builders, both doctests) all behave correctly under test; the only defect is an inaccurate, syntactically-broken default shown in the `syntax_colors/0` documentation.

## Bugfixes

#### `CSCAN-ansi-syntax-colors-doc-default`
- **Finding ID** — `CSCAN-ansi-syntax-colors-doc-default`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/io/ansi.ex:69-77` (the `@doc` for `syntax_colors/0`)
- **Issue** — The documented "The default is:" keyword list (a) is missing a comma after `atom: :cyan` (lines 70–71), making the shown literal invalid Elixir, and (b) lists only 6 keys while the actual runtime default has 13 (`binary`, `list`, `map`, `tuple`, `variable`, `call`, `operator` are all absent from the doc).
- **Class** — Documentation mismatch.
- **Impact** — A user consulting this doc to override `:ansi_syntax_colors` is shown a default that does not match reality and cannot be copy-pasted: pasting the snippet as-is yields a `SyntaxError` because `atom: :cyan boolean: :magenta` is not valid keyword syntax. Misleads configuration and confuses which types are actually colorized.
- **Evidence** — `verified`. Running the accessor shows the true default:
  ```
  $ elixir -e 'IO.inspect(IO.ANSI.syntax_colors(), limit: :infinity)'
  [atom: :cyan, binary: :default_color, boolean: :magenta, charlist: :yellow,
   list: :default_color, map: :default_color, nil: :magenta, number: :yellow,
   string: :green, tuple: :default_color, variable: :light_cyan,
   call: :default_color, operator: :default_color]
  ```
  This has 13 entries vs. the doc's 6, and the doc source (lines 70–71) reads `atom: :cyan` / `boolean: :magenta,` with no separating comma.
- **Suggested fix** — Add the missing comma and complete the list to match the real default (or replace with a short prose note pointing at the runtime value rather than an out-of-date literal).

## Risks / caveats
- Both published doctests (`format/2` at line 285, `format_fragment/2` at line 306) were executed and reproduce their documented output exactly.
- `String.to_unsafe_atom/1` (used at lines 164, 198, 201, 204) is a real function in this toolchain and the module compiles cleanly (`elixirc` exit 0), so it is not a defect despite `function_exported?/3` reporting `false` for it.
- All ANSI code numbers (fonts 11–19, colors 30–37/90–97/40–47/100–107, `normal`=22, aliases `inverse`/`reverse`=7, etc.) match the standard/upstream values; spot-checked `light_red()` → `"\e[91m"` and `font_3()` → `"\e[13m"`.

### Ruled out
- **`do_format/5` reset logic (lines 315–342)** — Suspected a wrong/missing `reset()` append. Verified correct: reset is appended only when `emit?` is `true` *and* an atom conversion flipped `:maybe`→`true` via `!!append_reset`; `format_fragment` (seeded `false`) never appends, and pure-text input returns `acc` unchanged, matching the doc "when a conversion is performed." Basis: `verified` (both doctests plus the trace of `format(:red, true)` producing a trailing `"\e[0m"`).
- **Atom-clause ordering vs. `not is_list` clause (lines 319–330)** — Suspected an atom could fall through to the binary/char clause. Ruled out: `emit?` is guarded `is_boolean`, so one of the two `is_atom` clauses always matches an atom before line 328 is reached. Basis: `asserted` from the guard, corroborated by the invalid-atom path raising `ArgumentError` as documented (`verified`: `format([:nope], true)`/`format([:nope], false)` both raise).
- **Improper-list / mixed chardata handling** — Checked `[?a | "bc"]`-style improper lists and integer/binary mixes thread through `rem`/`acc` correctly. Basis: `verified` (traced and ran `format_fragment([:bright, ~c"Word"], true)`).
- **`cursor/2` "line 0 / column 0 = top left" doc (line 240)** — Emits `\e[0;0H`; terminals are 1-based so this is slightly loose, but it is the long-standing upstream wording and reflects the literal emitted string, not a code defect. Basis: `asserted`.

