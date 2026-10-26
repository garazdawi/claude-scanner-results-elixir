# Cluster cross-file findings

#### `CSCAN-cluster-atom-escape-length-dos`

- **Finding ID** — `CSCAN-cluster-atom-escape-length-dos`
- **Severity** — Low
- **Confidence** — medium
- **Files involved**
  - `lib/code/normalizer.ex:569-579` (and the reach path at `:20-32`, `:256-258`)
  - `lib/code/identifier.ex:77-83`, `:116-168` (esp. `:141-142`)
  - `lib/string.ex:3025-3028`, `:3040-3042` (out-of-cluster sink definition)
- **Finding shape** — Other: a length-bounding safety valve exists in file A but is bypassed by file B, which then feeds the unbounded result into a hard-capped sink → crash (a "safe because a bound exists / caller checks" split).
- **Evidence**
  - `identifier.ex` exposes `escape/4` whose 3rd argument is a `limit` (`def escape(binary, char, limit \\ :infinity, …)`, `identifier.ex:79`). Its output is *length-amplifying*: every invisible/bidi codepoint is rewritten to a 6-byte `\uHHHH` sequence (`identifier.ex:141-142`), backslash to `\\` (`escape_map/1`, `:179`), etc. So a valid N-codepoint atom can become up to ~6·N codepoints.
  - `normalizer.ex` escapes atom literals by round-tripping through that function with the default (unbounded) `limit`, then *reconstructs an atom*:
    ```elixir
    defp maybe_escape_literal(string, %{escape: true}) when is_binary(string) do
      {string, _} = Code.Identifier.escape(string, nil)   # normalizer.ex:570  (limit = :infinity)
      IO.iodata_to_binary(string)
    end
    defp maybe_escape_literal(atom, %{escape: true} = state) when is_atom(atom) do
      atom |> Atom.to_string() |> maybe_escape_literal(state) |> String.to_unsafe_atom()  # :574-579
    end
    ```
  - The sink `String.to_unsafe_atom/1` is `:erlang.binary_to_atom(string, :utf8)` (`string.ex:3040-3042`), which raises `ArgumentError` on any binary exceeding the atom size cap — "The maximum atom size is 255 Unicode code points." (`string.ex:3028`).
  - Every atom literal in an AST reaches this path via `normalize_literal/3` (`normalizer.ex:256-258`), and `Macro.to_string/1` / `Code.quoted_to_algebra/2` drive it with `escape: true` by default (`macro.ex:1331-1333`, `code.ex:1576-1580`, `normalizer.ex:20-32`). Concretely, an atom of only ~43 bidi codepoints (e.g. `:"…"` built from `\u{202E}` repeats) is a legal ≤255-codepoint atom, but its escaped form is 43·6 = 258 codepoints, so `String.to_unsafe_atom` raises and `Macro.to_string` crashes. Such an atom literal is producible from untrusted source via `Code.string_to_quoted/2` → `Macro.to_string/1` (a common lint/format/codegen pipeline).
- **What the per-file pass missed** — A `normalizer.ex`-only reviewer sees "escape a string, make an atom" and cannot know `Code.Identifier.escape` may sextuple the length; an `identifier.ex`-only reviewer sees `escape` faithfully honoring its `limit` contract (with `:infinity` meaning "caller opted out of bounding"); a `string.ex`-only reviewer sees `to_unsafe_atom` correctly enforcing 255. The crash exists only in the composition, where the normalizer opts out of the bound and then hands the amplified output to the 255-capped atom constructor.
- **Suggested fix** — In `normalizer.ex:574-579`, avoid forcing the escaped atom back through `binary_to_atom`. Either keep the escaped value as a binary and adjust the downstream formatter branch that expects an atom, or bound/guard the length before atomizing, e.g.:
  ```elixir
  defp maybe_escape_literal(atom, %{escape: true} = state) when is_atom(atom) do
    escaped = atom |> Atom.to_string() |> maybe_escape_literal(state)
    if byte_size(escaped) <= 255 * 4, do: String.to_unsafe_atom(escaped), else: atom
  end
  ```
  (The precise guard should use codepoint length; the point is to stop feeding an unbounded transform into the 255-codepoint atom sink.)

---

Note on scope: I also traced `String.to_unsafe_atom` on BEAM-derived variable names in `typespec.ex:404-407` (`erl_to_ex_var`, reachable from `fetch_specs/1`/`fetch_types/1` on an attacker-supplied `.beam` binary → atom-table exhaustion/crash). That is a real `to_unsafe_atom`-on-untrusted-input vector the folder threat model calls out, but it spans only one cluster file plus `string.ex`, so it falls outside the "two or more files *in this cluster*" requirement and is left for the per-file/`string.ex` passes rather than reported here as a cluster finding.
