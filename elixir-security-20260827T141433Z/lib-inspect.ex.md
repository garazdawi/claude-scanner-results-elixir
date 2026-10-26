# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/inspect.ex
<!-- sha256: 4c6984cf8a1f282d888dfb7476e823c3decac912498100effef6d81e114b2c8b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:05:35Z_

I have a fully verified finding. The default-opts trigger requires a list longer than `printable_limit` (4096) with an ASCII-printable prefix; the `:as_charlists` opt drops the length requirement. The crash bypasses the struct safety net. Let me write the report.

---

# Security review — `lib/elixir/lib/inspect.ex`

## 1. Summary
The `Inspect` protocol implementations are sound on the code-execution, atom-creation, and Trojan-Source/bidi fronts (all delegated escaping correctly neutralizes bidi/control characters), but `Inspect.List.inspect/2` has a partial-validation flaw that lets attacker-influenced list data crash `Kernel.inspect/2`/`IO.inspect/2`/`Logger` with default options, bypassing the documented inspect safety net.

## 2. Security

#### `CSCAN-inspect-charlist-conversion-crash`
- **Finding ID:** `CSCAN-inspect-charlist-conversion-crash`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/elixir/lib/inspect.ex:390-397`, `Inspect.List.inspect/2` — sink at line 392 (`IO.chardata_to_string(term)`)
- **Class:** Unclassified (denial-of-service via unhandled conversion exception; underlying cause is a bounded validation used as a totality guarantee)
- **Issue:** The charlist branch is gated by `List.ascii_printable?(term, printable_limit)`, which only inspects the **first `printable_limit` (default 4096)** elements. When that gate passes, the code calls `IO.chardata_to_string(term)` on the **entire** list. Any element **beyond** position `printable_limit` that is not a valid Unicode scalar — a value `> 0x10FFFF`, a surrogate `0xD800..0xDFFF`, a negative integer, or an improper tail — makes `List.to_string/1` raise `UnicodeConversionError`/`ArgumentError`. The bounded check says "safe to render as a charlist," but the unbounded conversion then fails. `Inspect.BitString` does not have this problem because it feeds the raw binary to `Identifier.escape/3` (total over all bytes) instead of converting first. Because a `List` is not a struct, it takes the unguarded `to_doc_with_opts/2` clause (`algebra.ex:453`), so the struct `try/rescue` safety net never runs; and even when the bad list is a *field of a struct*, the rescue handler re-renders the same fields and re-crashes, so the net is defeated there too. With the explicit `charlists: :as_charlists` option the `ascii_printable?` gate is short-circuited entirely, so **any** malformed list (any length) crashes.
- **Impact / threat:** An attacker who controls list-shaped data that a host later inspects or logs (e.g. a term from `:erlang.binary_to_term/1`, an Erlang-interop charlist, or a decoder that emits integer/codepoint lists) can force an unhandled exception in the inspecting process. Since `inspect` underpins logging and crash reporting, this turns a benign `Logger.info(inspect(x))` / `IO.inspect(x)` into a process crash and defeats the safety mechanism specifically built to keep inspect total during error formatting. Trust boundary crossed: untrusted deserialized/interop data → the compiler/runtime's public inspection API. It is availability-only and requires list-shaped input (binary web input does not trigger it), which caps severity at Medium.
- **Evidence:** `verified`.
  - Default opts, top-level `Kernel.inspect/2`:
    ```
    $ elixir -e 'bad = List.duplicate(?a, 4097) ++ [0x110000]; IO.puts(inspect(bad))'
    ** (UnicodeConversionError) invalid code point 1114112
        (elixir 1.21.0-dev) lib/list.ex:1261: List.to_string/1
        (elixir 1.21.0-dev) lib/inspect.ex:392: Inspect.List.inspect/2
        (elixir 1.21.0-dev) lib/inspect/algebra.ex:453: Inspect.Algebra.to_doc_with_opts/2
        (elixir 1.21.0-dev) lib/kernel.ex:2480: Kernel.inspect/2
    ```
  - `List.ascii_printable?(List.duplicate(?a,5000) ++ [0x110000], 4096)` returns `true` while `inspect/1` of the same value raises — confirming the gate/conversion mismatch.
  - Variants all crash: surrogate `0xD800`, negative `-1`, improper tail, nested-in-list, and **struct-wrapped with `safe: true`** (safety net bypassed). Short lists whose bad element is within the first 4096 do **not** crash (element-wise path). `charlists: :as_charlists` crashes on any bad list regardless of length. Analogous `Inspect.BitString` case (`"a"×5000 <> <<0xFF,0xFE>>`) returns normally.
  - Impact path: `Logger.info(inspect(bad))` raised `UnicodeConversionError`.
- **Suggested fix:** Make the branch total. Smallest change: guard the conversion so a failure falls through to the existing generic element-wise arm (`container_doc_with_opts(open, term, close, opts, &to_doc_with_opts/2, separator: sep)`) instead of propagating — e.g. wrap `IO.chardata_to_string(term)` in `try/rescue` (rescuing `UnicodeConversionError`/`ArgumentError`) and, on failure, render the list element-wise. Equivalently, convert only the validated first `printable_limit` elements (preserving the truncation `"..."` indicator) rather than the whole list.

### Ruled out
- **Bidi / Trojan-Source injection (strings, atoms, regex, keyword keys)** — `verified` not exploitable. `inspect("hello\u202Eworld")` → `"hello\u202Eworld"`, `inspect(:"he\u202Ello")` → `:"he\u202Ello"`, `inspect(~r/ab\u202Ecd/…)` → `~r/ab\u202Ecd/`. `Code.Identifier.escape/escape_char` (`code/identifier.ex:126-140`) explicitly escapes U+202A–202E/2066–2069/200E/200F/061C and other invisibles; ESC/`\e` and C0 controls are hex-escaped, blocking ANSI-escape injection too.
- **Atom-table exhaustion (CWE-770)** — no path creates atoms; only `Atom.to_string/1`, `Atom.to_charlist/1`, `Macro.inspect_atom/2` (all consume existing atoms).
- **Code execution** — no `Code.eval*`/`compile*`/`apply`. `__deriving__/2` is a compile-time macro whose inputs are the developer's own `@derive` options plus `Macro.struct_info!`, validated by `validate_option/4`; not an untrusted-data boundary.
- **Secret leakage via default struct inspect** — the default `Inspect.Any` printing all fields is documented behavior; this file *provides* the redaction mechanism (`:only`/`:except`/`:optional`), and the `__deriving__` filter (`filtered_fields`, `field_guard`) correctly drops `:except` fields and switches to the non-copyable `#Name<…>` form. Not a defect in this file.
- **Deep-nesting / breadth resource DoS** — `verified` bounded: a 500,000-deep nested list inspects to 405 bytes in ~15 ms under default opts, because `limit` (default 50) is a global element budget over the whole traversal. Unbounded only with caller-chosen `limit: :infinity`.
- **Quadratic/large-input blowup in `Regex.normalize`/`Identifier.escape`** — bounded input: regex sources are capped by PCRE (`~c"regular expression is too large"` at a few tens of KB), and string escaping is capped at `printable_limit` (default 4096). Not exploitable.
- **Crafted `%Regex{}` with non-binary `source`** — requires a caller to hand-build a struct violating the `%Regex{}` contract; and being a struct it is caught by the `to_doc_with_opts` rescue and re-rendered as a map. Not reachable through the documented API with untrusted data.
- **`Inspect.Any` second clause `:io_lib.format("~p", …)`** — normal protocol dispatch routes structs to the first `%module{}` clause and every built-in non-struct type to its own impl, so no ordinary untrusted term reaches this "native record" fallback; niche and not shown reachable.
- **PID/Port/Reference/Function/Integer/Float impls** — deterministic conversions of VM-internal data with no untrusted-content interpolation (the one raw interpolation, `mod.__RELATIVE__()` in `Inspect.Function`, is a compile-time source path, not runtime data).

## 3. Risks / caveats
- **Reproduction environment:** I could not build from the exact checkout (only `lib/` is present), so I reproduced against the installed `/opt/elixir` (Elixir 1.21.0-dev, the same dev line). Confidence that this is the code under review is high: the crash stack trace points precisely at `lib/inspect.ex:392`, matching the `IO.chardata_to_string(term)` call in this file. Residual risk that the installed build differs from this file in unrelated ways is low.
- **Exploitability is data-shape dependent:** the default-opts crash needs list-shaped untrusted input longer than `printable_limit` with a printable prefix and a later invalid element. Common web inputs (binaries, JSON→maps) do not trigger it; the realistic sources are `binary_to_term`, Erlang interop charlists, and decoders emitting integer lists. I rated it Medium accordingly rather than High.
- **Not exhaustively enumerated:** I did not prove the second `Inspect.Any.inspect/2` clause is fully unreachable for every exotic Erlang term type; I found no dispatch path to it with untrusted data but did not brute-force all term shapes.

