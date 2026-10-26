# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/record.ex
<!-- sha256: e7e2d219936a71894f2bdb6767593464a12d0bc41de821bf9228150b7fc655e1 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:42:28Z_

# Review: `record.ex`

- **Summary:** The public record macros (`defrecord`/`defrecordp` and their generated `name/0,1,2` accessors), the `is_record` guards, and the runtime `__keyword__` conversion path all behave correctly under valid, invalid, duplicate-key, and match-context input; the only genuine defect is an internally inconsistent documentation example.

## Bugfixes

#### `CSCAN-record-moduledoc-age-default-mismatch`
- **Finding ID:** `CSCAN-record-moduledoc-age-default-mismatch`
- **Severity:** Low
- **Confidence:** high
- **Location:** `record.ex:195` (example in `defrecord/3` `@doc`), with the mismatching comments at `record.ex:205` and `record.ex:212`.
- **Issue:** The example defines the record with a **string** default (`Record.defrecord(:user, name: "meg", age: "25")`) but the accompanying result comments show an **integer**: `record = user() #=> {:user, "meg", 25}` and `user(record, age: 26) #=> {:user, "meg", 26}` (the latter comment is consistent because it overrides age; the `user()` comment is not). With the default as written, `user()` yields `{:user, "meg", "25"}`, not `{:user, "meg", 25}`.
- **Class:** Documentation mismatch.
- **Impact:** A reader copying the example is told the default field value is the integer `25` when it is actually the string `"25"`; harmless but misleading, and it makes the doc internally contradictory (string default vs integer result).
- **Evidence:** `verified` — ran the example verbatim:
  ```
  $ elixir -e 'defmodule User do require Record; Record.defrecord(:user, name: "meg", age: "25"); def a, do: user() end; IO.inspect User.a()'
  user()  ... : {:user, "meg", "25"}
  ```
  The code returns `{:user, "meg", "25"}`, contradicting the `#=> {:user, "meg", 25}` comment.
- **Suggested fix:** Change the definition in the example to `age: 25` (integer), matching the `#=>` comments — the simplest edit that makes the example self-consistent.

## Ruled out

- **Double-escaping of non-literal field defaults** (`fields/2` at line 344 escaping, then `create/4` re-escaping at line 426) — `verified`: records with `{1,2,3}`, `[1,2,3]`, and `%{x: 1}` defaults all produce the correct runtime value (`{:r3, {1, 2, 3}}` etc.), so net escaping is exactly once. No defect.
- **Truthiness trap in `error_on_duplicate_record/2`** (`if defined_arity` when `Enum.find(0..2, …)` can return `0`) — `asserted` from language semantics and `verified` by construction: `0` is truthy in Elixir (only `nil`/`false` are falsy), so an existing `name/0` still correctly triggers the raise. No off-by-one.
- **`join_keyword/3` size-mismatch count** returning `length(acc) + length(rest_fields)` — `verified`: passing `{:user, "meg"}` (too few) to the keyword-conversion path raises `"expected argument to be a :user record with 2 fields, got: {:user, \"meg\"}"`; the reported count equals the total field count and the error is accurate.
- **Duplicate field key in `create`/`update`** — `verified`: `user(name: 1, name: 2)` returns `{:user, 1, "25"}` (first wins, via `Keyword.pop` removing all entries) and `user(r, name: 1, name: 2)` returns last-wins; both are edge cases of caller misuse, no crash or misleading error.
- **`build_update/2` descending sort comparator (`right <= left`)** — `asserted`+`verified`: multi-field updates (`user(r, name: 1, age: 2)`) set every field correctly; each index is independent so ordering is a determinism choice, not a correctness issue.
- **`find_index` base offset (1 for `index/3` vs 2 for `get`/`update`)** — `verified`: `user(:name)` returns `1` (correct 0-based/`elem`-compatible index; tag at 0) while `get` generates `:erlang.element(2, …)` (correct 1-based Erlang position); both resolve to the same field. Intentional and consistent.
- **`update` inside a match** — `verified`: raises `"cannot invoke update style macro inside match"` at compile time as documented; match-context creation (`user(name: n) = rec`) correctly binds via wildcard-filled defaults.
- **`__access__/4` non-literal runtime argument** (`user(runtime_var)`) — `verified`: falls through to the quoted `__keyword__` path and converts a real record to a keyword list; a non-record runtime value raises the documented `"expected argument to be a literal atom, literal keyword or a :user record, got runtime: …"`.
- **`extract`/`extract_all`** — out of scope: they only guard argument types and delegate to `Record.Extractor` (a different file).

## Risks / caveats
- `extract/2`, `extract_all/1` bodies live in `Record.Extractor` and were not reviewed here; the doctest at line 95 depends on that module plus an OTP header file and was not executed.
- The escaping mechanism (`fields/2` + `create/4` interacting through nested `quote`/`unquote` in `defrecord`) was validated behaviorally rather than by tracing every quoting level; I confirmed net-correct output for scalar, tuple, list, and map defaults, but exotic AST-valued defaults were not exhaustively enumerated.

