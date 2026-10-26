# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/uri.ex
<!-- sha256: 12ffc79551caad9484b8c98c885faf3ab5db1d84bed9fe51307071cd274f8c91 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T00:02:23Z_

All documented paths behave correctly. I found one genuine defect (verified) on the deprecated dict branch. Report follows.

## Summary

`URI` is a faithful and correct RFC 3986 implementation — every doctest and the full RFC 3986 §5.4 merge/dot-segment suite pass — with one live defect: the deprecated struct path of `decode_query/3` constructs the wrong module atom and always crashes.

## Bugfixes

#### `CSCAN-uri-decode-query-dict-wrong-atom`
- **Finding ID** — `CSCAN-uri-decode-query-dict-wrong-atom`
- **Severity** — Low
- **Confidence** — high
- **Location** — `uri.ex:255-256`, `decode_query_into_dict/3` (reached from the `decode_query/3` struct clause at `uri.ex:218`)
- **Issue** — `String.to_unsafe_atom("Dict")` produces the bare Erlang atom `:Dict` (module `:Dict`, which does not exist), not the Elixir module alias `Dict`, whose atom is `:"Elixir.Dict"`. The following `dict_module.put(dict, key, value)` therefore calls `:Dict.put/3`, which is undefined, so this branch raises `UndefinedFunctionError` for every input. The target module actually exists and is callable in this build (`function_exported?(Dict, :put, 3) == true`), so the branch was meant to work — only the atom is wrong.
- **Class** — Unclassified (wrong module-atom construction)
- **Impact** — A caller who passes a struct as the second argument to `decode_query/3` — a usage the code deliberately branches on and warns is *deprecated* (i.e. still expected to function), not *removed* — gets `** (UndefinedFunctionError) function :Dict.put/3 is undefined (module :Dict is not available)` instead of a populated result. The deprecation warning misleads: the "deprecated" path is in fact entirely broken. This is a niche/deprecated surface, hence Low.
- **Evidence** — `verified`.
  - `String.to_unsafe_atom("Dict")` → `:Dict`; `:code.which(:Dict)` → `:non_existing`; while `:code.which(Dict)` → `.../Elixir.Dict.beam` and `function_exported?(Dict, :put, 3)` → `true`.
  - Recompiling this exact file and calling `URI.decode_query("a=1&b=2", struct(FakeDict))` prints the deprecation warning, then: `RESCUE: UndefinedFunctionError - function :Dict.put/3 is undefined (module :Dict is not available)`. The stock `/opt/elixir` build reproduces the identical crash, so it is not an artifact of recompilation.
- **Suggested fix** — Construct the correct module atom or reference the module directly: replace `dict_module = String.to_unsafe_atom("Dict")` / `dict_module.put(...)` with `Dict.put(dict, key, value)` (or `String.to_unsafe_atom("Elixir.Dict")`). Simplest is to drop the `decode_query_into_dict` branch entirely, since `Dict` is long deprecated.
- **Scope caveat** — Passing a struct technically violates the `@spec` second argument `%{optional(binary) => binary}`. I report it rather than rule it out because the implementation has a dedicated clause that intentionally accepts structs and emits a "deprecated" (not "unsupported") warning, i.e. the code's own contract promises this path works; the validator can weigh the spec-violation angle.

## Risks / caveats

- Everything else I exercised is correct. Verified by recompiling the file under review and running:
  - **All module doctests** (`default_port`, `encode_query`/`decode_query`/`query_decoder` in both `:www_form` and `:rfc3986`, `char_*`, `encode`/`decode`/`encode_www_form`/`decode_www_form`, `new`/`new!`/`parse`, `append_query`/`append_path`, `merge`) — all match.
  - **RFC 3986 §5.4.1 + §5.4.2** merge suite (34 normal + abnormal cases incl. `..`/`.`, `//g`, `/../g`, `g;x=1/../y`) — all match, so `merge`/`merge_paths`/`remove_dot_segments`/`join_reversed_segments` are correct.
  - Edge cases: IPv6 host bracket stripping (`parse`/`new`), userinfo-with-`@`, empty/`//` authority, oversized port, `append_path` `//`/leading-slash/trailing-slash logic, `append_query` `&`-boundary logic, `to_string` absolute-path-with-host guard and IPv6 re-bracketing/default-port omission, truncated percent escapes (`%`, `%4`), missing `=` in query pairs, list-key/value rejection — all behave as documented.
- I could not diff against a pristine upstream copy offline; parity claims above rest on the executed RFC/doctest oracles, not on comparison to canonical source.

### Ruled out
- **`decode_query/3` deprecated-`dict` `String.to_unsafe_atom` path being harmless** — not ruled out; promoted to the finding above (`verified` crash).
- **`merge`/dot-segment threading, clause ordering, `:+` base-marker handling** — no defect; `verified` against the full RFC 3986 §5.4 vector set (57 cases across two scripts), all exact matches.
- **`parse`/`new`/`uri_from_map` port/scheme/host normalization, IPv6 handling, `nilify`/`nilify_query`** — no defect; `verified` across schemes, ports, IPv6, empty authority, and the documented `{:error, part}` shapes.
- **`encode`/`decode`/`encode_www_form`/`decode_www_form`/`encode_unreserved`/`unpercent` byte-level and multibyte (`й`) handling, truncated escapes** — no defect; `verified` (correct `%20`/`+` distinction, UTF-8 byte encoding, no crash on malformed `%`).
- **`String.Chars.URI.to_string` authority assembly, absolute-path guard, default-port omission, IPv6 bracketing** — no defect; `verified`.
- **`default_port/1,2` via `:elixir_config.get/2`** — safe; uses the crash-safe two-arg form with a `nil` default (`verified` returns `nil` for unknown scheme, round-trips after `put`).

