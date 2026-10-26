# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io.ex
<!-- sha256: cf416b21d1eb96dd73bf29e687eb97e69842af12ae3b45e7ab8828f7b1d5236d -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:21:54Z_

# IO module bug-hunt review

## Summary
The public API of `IO` is almost entirely correct; the only genuine defect is in the deprecated `IO.read(device, :all)` path, which queries device options on the un-mapped `:stdio`/`:stderr` alias and therefore can never return a charlist for a list-mode standard device.

## Bugfixes

#### `CSCAN-io-read-all-unmapped-getopts`
- **Finding ID:** `CSCAN-io-read-all-unmapped-getopts`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/elixir/lib/io.ex:184` — `read(device, :all)`
- **Issue:** The empty-input branch does `:io.getopts(device)` on the raw `device` instead of `map_dev(device)`. Every other read/write helper in the file maps `:stdio`→`:standard_io` and `:stderr`→`:standard_error` first. `:io.getopts(:stdio)` (and `:stderr`) is not a registered process name and returns `{:error, :arguments}`, which fails the `with [_ | _] = opts <- ...` match and drops to `else -> ""`. So for the two atom shortcuts the "is this device in list mode?" test is dead code — `IO.read(:stdio, :all)` on an empty list-mode (`binary: false`) standard device returns the binary `""` when it should return the charlist `~c""`.
- **Class:** Pattern match on the wrong shape (device alias not normalized before `:io.getopts/1`).
- **Impact:** A caller using the deprecated `IO.read(:stdio, :all)` / `IO.read(:stderr, :all)` against a list-mode device gets `""` (binary) instead of `~c""` (charlist) at end-of-input, so pattern matches / type assumptions keyed on charlist output silently break on the empty case. Only affects the deprecated `:all` form and only the `:stdio`/`:stderr` shortcuts (a raw pid or user-registered device name works, because `getopts` succeeds there). Deprecated (`TODO: Remove me on v2.0`), narrow, and empty-input-only, hence Low.
- **Evidence:** `verified` — `:io.getopts(:stdio)` → `{:error, :arguments}` and `:io.getopts(:stderr)` → `{:error, :arguments}`, while a list-mode pid device returns a proper opts keyword list (see run output above). The `{:error, :arguments}` value cannot match `[_ | _]`, so the `~c""` branch is unreachable for the aliases and the code always yields `""`.
- **Suggested fix:** Map the device first: `with [_ | _] = opts <- :io.getopts(map_dev(device)), false <- Keyword.get(opts, :binary, true) do ~c"" ...`, matching the rest of the module.

## Risks / caveats
- I reviewed only `lib/elixir/lib/io.ex`; defects in the called modules (`IO.Stream`, `Inspect.Algebra`, `:elixir_errors`, `String`, `StringIO`) are out of scope and untested here.
- The `getn/2` and `stream/2` overloads reject out-of-range integers (`0`, negatives, `getn(_, 0)`) with `FunctionClauseError`, but only for arguments that already violate the documented `pos_integer`/`non_neg_integer` specs, so I did not treat those as findings.
- I could not exercise the exact live `IO.read(:stdio, :all)` empty-stdin case interactively; the finding is established from the verified `:io.getopts/1` behavior on the alias atoms plus the surrounding `with` logic, which fully determines the branch taken.

### Ruled out
- **`binread(device, :line)` CRLF normalization** — I suspected `:file.read_line/1` would leave `\r\n` intact, contradicting the doc's "normalized to \n". `verified`: on a real file opened `[:read, :binary]`, `:file.read_line` returned `<<108,49,10>>` ("l1\n") while `:file.read` returned raw `...13,10...`; `:file.read_line` does normalize CRLF, so the doc is correct.
- **`iodata_empty?/1` totality/correctness** — checked `[]`, `[""]`, `[1,2|<<3,4>>]`, nested-empty, improper-empty (`[[] | <<>>]`), improper-nonempty (`[[] | <<0>>]`), and a zero byte `[0]`. `verified`: all return the correct boolean, including `[0]` → `false` (a zero byte is content, matching `iodata_length`); the `head and tail` recursion short-circuits correctly.
- **`warn/2` clause dispatch (Macro.Env / keyword / `[]` / stacktrace)** — the keyword clause `[{_,_}|_]` and the stacktrace clause `[{_,_,_,_}|_]` are disjoint (2- vs 4-tuples) and `[]` has its own clause ahead of the stacktrace clause. `verified`: full keyword, empty keyword, and keyword-without-`:file` all returned `:ok` and emitted the expected diagnostic.
- **`read(device, :eof)` / `getn_eof` / `wrap_eof` accumulation** — `verified` via `StringIO`: multi-line, trailing-no-newline, CRLF, and empty inputs all returned correctly ordered joined output, and empty returned `:eof`; this (non-deprecated) path correctly uses `map_dev`, unlike the `:all` path above.
- **`binread(device, :eof)` empty-vs-data** — `verified`: `"abc"` → `"abc"`, empty → `:eof`.

