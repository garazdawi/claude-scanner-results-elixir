# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/module/types/helpers.ex
<!-- sha256: a7d911c43fdf9ef0d816ef4160c9043ff361ea6dba8a167f1b18d66f6ca374f8 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:00:11Z_

I've read the entire file and exercised the one path with any dynamic behavior. Here is the report.

---

## Summary

`Module.Types.Helpers` is an internal (`@moduledoc false`) formatting/AST-helper module for the type checker whose inputs are compiler-internal AST fragments and type descriptors; it contains no atom-creation, eval, command, file-write, deserialization, or injection sinks reachable through its documented API, so I found no meaningful security issues.

## Security

`No findings`

### Ruled out

- **`struct.__info__(:struct)` dynamic dispatch in `prewalk_expr_to_string` (helpers.ex:314–318).** The struct name comes from a `%Struct{}` AST node in the analyzed source, so it is dispatch on an externally-derived atom (Protocol/behaviour-dispatch class). Basis: **verified**. The call is wrapped in `try/rescue _ -> node`, and I drove it directly: `elixir -e 'IO.puts(Module.Types.Helpers.expr_to_string({:%, [], [DefinitelyNotARealModule123, {:%{}, [], [a: 1, b: 2]}]}))'` printed `%DefinitelyNotARealModule123{a: 1, b: 2}` (the `UndefinedFunctionError` was caught and the node returned unchanged), while a real module (`URI`) collapsed to defaults. `__info__/1` is compiler-generated and reserved (not user-overridable), it can only resolve to a struct module already compiled/loaded for the source to reference it, and any failure is rescued — no code execution or crash is added here.
- **Atom creation.** None. `collect_traces` uses `Atom.to_string(var_name)` (atom→string, the safe direction); there is no `String.to_atom`, `List.to_atom`, `:erlang.binary_to_atom`, or `Module.concat` anywhere in the file (full-file read).
- **Path/file operations.** `format_trace` calls `Path.relative_to_cwd(trace.file)` (helpers.ex:230) purely to build a display string; no `File.*`, no path is used to read/write/open anything, so no traversal/TOCTOU surface.
- **Unbounded/quadratic resource use.** The recursions (`unpack_list/2`, `unpack_match/2`, `zip_map_reduce/5`) and tree walks (`Macro.prewalk`, `Macro.to_string`, `indent/2`, the `Enum.sort_by/dedup` in `collect_var_traces`) are all bounded by the depth/size of already-parsed AST; none introduces a new super-linear blowup on external input beyond what parsing the source already cost.
- **String interpolation.** `format_var`, `integer_to_ordinal`, and `format_hints` interpolate values into human-readable diagnostic text only; the sink is a compiler warning/error message, not a downstream interpreter (query/shell/regex/markup), so there is no injection target.
- **Eval / command / deserialization / regex / shared-state sinks.** Absent — no `Code.eval*`/`compile*`, `System.cmd`/`:os.cmd`/`System.shell`, `:erlang.binary_to_term`, `Regex.compile`/`~r//`-from-input, `:ets`/`:persistent_term`/`Application.put_env`/`Process.put`, or `Base`/`URI` round-trip decoders in this file.

## Risks / caveats

- The module is `@moduledoc false`; nothing here is Elixir public API. I applied the review's "documented public API" scope to the `@doc`-annotated functions (notably `collect_traces/2`, flagged in-source as consumed by language servers). Even under that reading, all inputs are compiler-internal AST/type-descriptor terms produced during analysis of the source being compiled — the language server consumes this module's *output* rather than feeding it attacker-shaped terms — so the untrusted-source-code boundary does not put hostile data into an unguarded sink here.
- I verified the only dynamic-dispatch path (`struct.__info__`) by direct execution against the source-built toolchain (Elixir 1.21.0-dev / OTP 29). The remaining conclusions rest on a complete read plus sink enumeration rather than execution, because the other functions are pure formatting/recursion with no external-effect sink to reproduce.
- Deeply nested adversarial AST could in principle grow stack usage in `Macro.prewalk`/`Macro.to_string`, but such AST must first survive the parser (a different module) that already bounds nesting; this file adds no new limit-bypassing recursion, so I did not treat it as a finding of this file.

