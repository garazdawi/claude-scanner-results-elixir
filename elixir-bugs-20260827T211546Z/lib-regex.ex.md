# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/regex.ex
<!-- sha256: b3804f759fcd8aaaae6e5031dcdde65be4299cc83a88f583a72944ce273e95b4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:45:09Z_

All doctests pass and the split/parts semantics are correct. I've confirmed one solid defect class. Let me write the report.

## Summary

`Regex` is largely correct — all 49 doctests pass and split/replace/capture semantics hold on the cases I drove — but the string-replacement compiler crashes with an opaque internal error on any `\g{...}` whose contents aren't a bare integer, a path reachable through the documented `Regex.replace/4` entry point.

## Bugfixes

#### `CSCAN-regex-g-brace-nonnumeric-crash`
- **Finding ID** — `CSCAN-regex-g-brace-nonnumeric-crash`
- **Severity** — Medium
- **Confidence** — high
- **Location** — `lib/regex.ex:867-870`, `precompile_replacement/1` (the `<<?\\, ?g, ?{, rest::binary>>` clause), reached from `replace/4` at line 851/855.
- **Issue** — This clause assumes the bytes after `\g{` are decimal digits followed by `}`. It calls `pick_int/1` (which only consumes `?0..?9`) and then destructures `{ns, <<?}, rest::binary>>}` and `List.to_integer(ns)`. Any `\g{...}` that is not exactly `\g{<digits>}` breaks one of those two assumptions: `\g{}` → `List.to_integer([])` raises `ArgumentError`; `\g{1` (unterminated) / `\g{12x}` / `\g{name}` / `\g{ 1}` → the `<<?}, rest>>` match fails and raises `MatchError`. There is no clause or `rescue` to turn this into a clean error.
- **Class** — Missing clause / non-total function (`MatchError`/`ArgumentError` on an input shape the parser doesn't handle).
- **Impact** — A caller who passes a replacement string containing a malformed or named `\g{...}` group — e.g. a natural attempt to reference a *named* capture `~r/(?<n>\d)/ … "[\g{n}]"`, which the docs do not support but a user will reasonably try, or a replacement template built from external/user input — gets an unhandled exception whose message (`no match of right hand side value: {~c"1", ""}` or `not a textual representation of an integer`) names nothing the caller can act on and does not mention the replacement string. On attacker-influenced replacement text this is an availability crash.
- **Evidence** — `verified`:
```
$ elixir -e 'Regex.replace(~r/(?<n>\d)/, "a1b", "[\\g{n}]")'
** (MatchError) no match of right hand side value:
    {[], "n}]"}
    (elixir 1.21.0-dev) lib/regex.ex:868: Regex.precompile_replacement/1
```
```
repl=\g{} : {ArgumentError, "...not a textual representation of an integer..."}
repl=\g{1 : {MatchError, "...{~c\"1\", \"\"}..."}
repl=\g{12x}: {MatchError, "...{~c\"12\", \"x}\"}..."}
```
(The documented forms `\N` and `\g{N}` with a bare integer work correctly and are unaffected.)
- **Suggested fix** — Make the `\g{` clause total: after `pick_int`, verify `ns != []` and that the remainder actually starts with `}`; otherwise either treat the sequence literally or `raise ArgumentError, "invalid capture reference #{inspect ...} in replacement"`. Minimal shape:
```elixir
defp precompile_replacement(<<?\\, ?g, ?{, rest::binary>>) when byte_size(rest) > 0 do
  case pick_int(rest) do
    {[_ | _] = ns, <<?}, rest::binary>>} -> [List.to_integer(ns) | precompile_replacement(rest)]
    _ -> raise ArgumentError, "invalid \\g{...} reference in Regex replacement"
  end
end
```

### Ruled out
- **`to_embed/2` raises on a `[:dotall]`-only compiled regex (and silently drops it under `strict: false`)** — reachable via `Regex.compile("foo", [:dotall])` then `Regex.to_embed/1`, which raises `ArgumentError` because `embeddable_modifiers/3` only recognizes the adjacent pair `[:dotall, {:newline, :anycrlf}]` (verified: raises; `strict: false` returns `"(?-imsx:foo)"`, losing dotall). Ruled out as *documented*: the `to_embed/2` `@doc` explicitly defines `'s'` as the pair `:dotall, {:newline, :anycrlf}`, and documents that under `strict: false` "the pattern will be returned as though any offending options had not be used." Basis: `verified`.
- **`match?/scan/run/split/replace` return-shape handling** — I drove no-names `named_captures` (`%{}` on match, `nil` on no-match), `capture: :all_names`/`:none`, zero-width and empty-regex `replace`/`split`, function replacements with arity exceeding available captures, and out-of-range numeric `\N`; every case returned the documented shape without crashing. Basis: `verified`.
- **OTP-version comparison `:erlang.system_info(:otp_release) >= [?2, ?8]` in `safe_run/3`** — lexicographic charlist compare misorders a hypothetical `"100"` release (`~c"100" < ~c"28"`), but on the current OTP 29 it evaluates `true` and, more importantly, freshly compiled patterns carry a `Reference` (not a binary) as the 5th `:re_pattern` element, so the recompile-from-source branch is not taken for normal patterns (verified: "5th not binary -> direct run path"). No reachable defect on any released OTP. Basis: `verified`.

## Risks / caveats
- I confirmed behavior by running against the in-repo build (Elixir 1.21.0-dev / Erlang OTP 29). I did not exercise OTP ≤27 pattern-serialization paths (`safe_run/3` recompile branch, `import/1`, `__escape__/1`) on an actual old release; those are version-shims outside this environment.
- `__escape__/1` and `__import_pattern__/1` are `@doc false` compiler-internal entry points (sigil expansion), so I excluded them per the internal-interface scope rule; a defect there would still affect every `~r//` literal and may warrant a separate look.
- The `\g{...}` finding is the only reachable crash I found; the replacement mini-language (`precompile_replacement`/`apply_replace`/`get_index`) is otherwise total over the digit/`\\`/literal cases I tested.

