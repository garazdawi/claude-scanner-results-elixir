# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/file/stream.ex
<!-- sha256: 1d6b82eb01621a93d88f863d2d851f8dc995063446c081896f433a356898d821 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:07:15Z_

I have enough to write the report. Let me confirm the dead-clause reasoning once more against the observed output (UTF-32LE `FF FE 00 00` → left `<<0,0>>`; UTF-32BE correctly stripped), which is already verified above.

## Summary
The `File.Stream` streaming and counting paths are correct for line and byte modes, but `bom_length/1` misidentifies the UTF-32LE byte-order mark, silently leaving garbage bytes in a `:trim_bom` stream.

## Bugfixes

#### `CSCAN-stream-utf32le-bom-mislength`
- **Finding ID** — `CSCAN-stream-utf32le-bom-mislength`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/file/stream.ex:220-225`, `bom_length/1` (reached via `trim_bom/2` from `skip_bom_and_offset/3` and `count_raw_bom/2`)
- **Issue** — The UTF-32LE BOM is the 4 bytes `FF FE 00 00` (`255, 254, 0, 0`). There is no clause for it. Its leading two bytes `FF FE` match the earlier UTF-16LE clause `<<255, 254, _rest::binary>>` (line 222), so `bom_length/1` returns `2` and only two of the four BOM bytes are trimmed. Separately, clause `<<254, 255, 0, 0, _rest::binary>>` (line 224) is dead code: it holds the wrong byte sequence for any real BOM (`FE FF 00 00` is not a UTF-32 BOM — UTF-32LE is `FF FE 00 00`), and even a corrected `<<255, 254, 0, 0>>` clause placed here would be shadowed by the 2-byte UTF-16LE clause on line 222.
- **Class** — Missing clause / non-total function (clause ordering + missing case)
- **Impact** — A caller doing `File.stream!(path, [:trim_bom])` (or `Enum.count/1` on such a stream) over a UTF-32LE-encoded file gets the leftover `<<0, 0>>` bytes prepended to the first streamed chunk instead of a clean BOM-stripped stream — silent data corruption of the first record. The byte `count` stays internally consistent with the stream (both trim only 2 bytes), so the corruption is invisible to length checks. Rare encoding and best-effort feature, hence Low.
- **Evidence** — `verified`. With `printf '\xFF\xFE\x00\x00ABCD' > u32le.txt` and `printf '\x00\x00\xFE\xFFABCD' > u32be.txt`:
  ```
  {"/tmp/u32le.txt", [<<0, 0, 65, 66, 67, 68>>]}   # UTF-32LE: 00 00 leaked, only 2 bytes trimmed
  {"/tmp/u32be.txt", ["ABCD"]}                      # UTF-32BE: correctly trimmed (clause 4 reachable)
  ```
  The UTF-32BE result confirms clause 4 (`<<0, 0, 254, 255, _>>`) is reachable and correct, isolating the defect to the missing/misplaced UTF-32LE case.
- **Suggested fix** — Add a correct UTF-32LE clause *before* the UTF-16LE clause and drop the dead one:
  ```elixir
  defp bom_length(<<0, 0, 254, 255, _rest::binary>>), do: 4   # UTF-32BE
  defp bom_length(<<255, 254, 0, 0, _rest::binary>>), do: 4   # UTF-32LE (must precede UTF-16LE)
  defp bom_length(<<239, 187, 191, _rest::binary>>), do: 3    # UTF-8
  defp bom_length(<<254, 255, _rest::binary>>), do: 2         # UTF-16BE
  defp bom_length(<<255, 254, _rest::binary>>), do: 2         # UTF-16LE
  defp bom_length(_binary), do: 0
  ```
  (Placing the 4-byte forms first guarantees they aren't shadowed by the 2-byte forms.)

## Risks / caveats
- I verified line-count vs. enumeration parity (no-EOL, empty, blank-only files), byte-count parity (with BOM and with `read_offset`), and that `read_offset`/`:trim_bom` left in `modes` are silently ignored by `:file.open` (so `Collectable.into` on such a stream does not crash — ruled out). Remote-node paths (`__open__/2` via `:erpc`, byte `count` via `:erpc.call(node, File, :stat, …)`) were reviewed by reading only, not executed — no reachable defect found there, but they were not run.
- `bom_length` uses a catch-all clause, so the defect is a wrong value, never a crash.

### Ruled out
- **`__build__` `with/1` has no `else`** — On valid input the `with` returns an unused `true`/`false`; the return value is discarded and the struct is built afterward, so the absent `else` is harmless. Basis: `verified` (streams built with and without `read_offset` behave correctly).
- **`trim_bom(device, false)` returns a character count `1` as its second element while `trim_bom(device, true)` returns a byte position** — the `false`-path second element is never consumed (`skip_bom_and_offset` takes `elem(0)`; `count_raw_bom` always calls the `true` path), so the inconsistency is unreachable. Basis: `asserted` (traced both call sites — the only two callers of `trim_bom/2` are lines 179 and 188; 179 forces `true`, 188 takes `elem(0)`).
- **Byte `count` returns `{:error, __MODULE__}` for a 0-size file instead of `{:ok, 0}`** — forces `Enum` to fall back to enumeration, which yields `0`; slower but not wrong. Basis: `verified` (empty-file count returns `0`).

