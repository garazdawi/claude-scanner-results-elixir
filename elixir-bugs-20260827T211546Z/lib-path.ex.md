# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/path.ex
<!-- sha256: b6144de2c362f1167f29efc86511520e6a29d2221736bd74df0d88f3a3f10649 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:34:59Z_

## Summary

The `Path` module's `join/2` fails to normalize the right-hand operand before matching it, so passing chardata (a charlist) equal to `"/"` produces a spurious `"."` component that the binary form correctly elides.

## Bugfixes

#### `CSCAN-path-join-chardata-root-slash`
- **Finding ID** — `CSCAN-path-join-chardata-root-slash`
- **Severity** — Low
- **Confidence** — high
- **Location** — `path.ex:676-687`, `join/2` and `do_join/3`
- **Issue** — `join/2` normalizes only `left` via `IO.chardata_to_string(left)` (line 677) and passes `right` to `do_join/3` untouched. The special clause `defp do_join(left, "/", os_type)` (line 682) that collapses a bare/trailing separator matches only the **binary** `"/"`. When `right` is chardata representing `"/"` (e.g. `~c"/"` or `[?/]`), that clause is skipped and control falls to the general clause (line 686), which calls `relative(right)`. `relative("/")` resolves to `"."` (via `unix_pathtype(~c"/") -> {:absolute, "."}`, line 300), so a `"."` component is appended instead of being removed.
- **Class** — Binary vs charlist vs string
- **Impact** — The module documents and demonstrates that its functions accept chardata (`t :: IO.chardata()`; the `Path.join("foo", ["bar", "fiz"])` example), so two equivalent inputs give different results: `Path.join("foo", "/") == "foo"` but `Path.join("foo", ~c"/") == "foo/."`, `Path.join("/", ~c"/") == "/."` (vs `"/"`), `Path.join("", ~c"/") == "."` (vs `""`). A caller that assembles a path from charlist fragments (directly or through `join/1`, which routes through `join/2`) gets a malformed path with a stray `/.` segment — wrong when compared, used as a key, or displayed.
- **Evidence** — `verified`:
```
$ elixir -e 'IO.inspect Path.join("foo", ~c"/"); IO.inspect Path.join("foo", "/")'
"foo/."
"foo"
$ elixir -e 'IO.inspect Path.join("/", ~c"/"); IO.inspect Path.join("", ~c"/"); IO.inspect Path.join("foo", [?/])'
"/."
"."
"foo/."
```
- **Suggested fix** — Normalize `right` the same way as `left` in `join/2`: add `right = IO.chardata_to_string(right)` before the `do_join(left, right, os_type)` call. Verified this keeps the chardata example correct (`Path.join("foo", ["bar", "fiz"]) == "foo/barfiz"`), since `do_join` then receives the already-flattened `"barfiz"`.

## Risks / caveats

- I ran every runnable doctest/example in the file (`join/1,2`, `safe_join`, `safe_relative`, `basename`, and the full `relative_to/3` example block including the `force: true` cases and divergent-prefix edges) against the locally built Elixir; all matched their documented outputs.
- The `.erl` compiler/Unicode-conformance obligations in the standing threat model do not apply to this file — `path.ex` is a pure path-string library that delegates OS-level parsing to `:filename`/`:filelib`; it performs no tokenization, identifier validation, or normalization.
- `safe_relative/2`, `safe_relative_to/2`, and `safe_join/2` delegate their traversal-safety guarantee to `:filelib.safe_relative_path/2` (OTP); I did not audit that OTP function, only that the wrappers pass arguments and translate `:unsafe -> :error` correctly.

### Ruled out
- **`absname/2` collapses interior `"./"` despite docs saying "no attempt is made to resolve `.`"** — `Path.absname("foo/./bar", "base")` returns `"base/foo/bar"` (interior `.` removed) while a trailing `.` is kept (`"base/foo/."`) and `..` is genuinely preserved (`Path.absname("../x", "bar") == "bar/../x"`). Basis: `verified`. This is the longstanding redundant-separator/`.`-segment normalization inherent to `:filename`-style path joining (`do_absname_join` line 151), and the documented contrast with `expand/1` concerns `..`/`~` resolution, which `absname` genuinely does not perform. Low enough impact and intended behavior; not a defect.
- **`join([])` raises `FunctionClauseError`** — no clause matches the empty list. Basis: `verified`. The `@doc` explicitly states "Raises an error if the given list of paths is empty," so this is the documented contract, not a bug.
- **`safe_join/2` `with` returning the unmatched `:error`** — when `safe_relative` returns `:error`, the `with` (no `else`) returns `:error`, which matches the `@spec {:ok, t} | :error`. Basis: `verified` (`Path.safe_join("foo", "../bar") == :error`). Correct.
- **`relative_to_forced` clause coverage for divergent/empty tails** — tested path-shorter, cwd-shorter, different-root, and both-relative cases; all produced correct `..`-prefixed results with no `FunctionClauseError`. Basis: `verified`.

