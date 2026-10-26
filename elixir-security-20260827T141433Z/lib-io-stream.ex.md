# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io/stream.ex
<!-- sha256: 8395113d478990bf6f24ced4c14ef9b611cb9e198ef68112e84f73c7091b0cfb -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io/stream.ex
<!-- sha256: 8395113d478990bf6f24ced4c14ef9b611cb9e198ef68112e84f73c7091b0cfb -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28. Neighbour `io.ex` read in full; the one execution axis (a live io-server replying `{:error, _}` to `put_chars`) was tried against the built Elixir toolchain._

## Summary

`IO.Stream` is a struct plus `Collectable`/`Enumerable` protocol implementations that delegate all real work to `IO.write`/`IO.binwrite`/`IO.each_stream`/`IO.each_binstream`. It contains no dangerous sinks (no eval, atom creation, path/file/command/network/crypto/deserialization) originating in this file, and I found no security issues reachable through its documented public API. The first-pass hedges about behaviour "outside this file" have been resolved by reading `io.ex` (the sole callgraph neighbour for every delegation) and by executing the one path that reading cannot settle. No findings.

## Security

`No findings`

Every conceptually-applicable Phase 1 class is empty here:

- **Code execution / compile-time execution / deserialisation / command execution / archive / network / crypto** — none of these primitives appear; the module only moves bytes to/from a caller-supplied IO device.
- **Atom creation** — no `String.to_atom`/`Module.concat`/`binary_to_atom`. The struct carries an already-resolved `device` term.
- **File / path handling** — no path is constructed or resolved here. The `device` is passed in by the caller via `IO.stream/2` (`io.ex:666-671`) / `IO.binstream/2` (`io.ex:707-712`), which run it through `map_dev/1` (`io.ex:850-855`); no data inside `stream.ex` derives a path.
- **Template / interpolation** — `IO.StreamError.message/1` uses `#{inspect(reason)}`. `inspect` renders an inert textual representation; it does not evaluate. Verified below.
- **Unicode / text** — the `raw` boolean merely selects `IO.each_binstream`/`IO.binwrite` (byte path) vs. `IO.each_stream`/`IO.write` (Unicode-translating path). No normalization/case-folding/confusable logic lives in this file; that is in the `IO`/`:unicode`/io-server layer. Confirmed by reading the delegated functions in `io.ex`.
- **Resource consumption** — `line_or_bytes` is passed straight through to `IO.each_stream` (`io.ex:822-834`) / `IO.each_binstream` (`io.ex:836-848`). Both public constructors already guard it to `:line | pos_integer` via `when line_or_… == :line when is_integer(...) and ... > 0` (`io.ex:667-669`, `io.ex:708-710`). Any unbounded-`:line`-read concern belongs to the `read`/`binread`/io-server layer (`io.ex:197-199`, `io.ex:246-251`), not this file.

### Ruled out

- **Collector swallows/mishandles write errors (`stream.ex:47-58`)** — *Basis: verified (executed).* The `{:cont, x}` clause returns the value of `IO.binwrite`/`IO.write`, which becomes the next accumulator; a non-`:ok` accumulator would fail to match `:ok, {:cont, x}` / `:ok, _` and raise. I chased both raw modes to ground:
  - `raw: true` → `IO.binwrite/2` **raises** on failure (`with {:error, reason} <- :file.write(...) do :erlang.error(reason)`, `io.ex:306-310`) — it can never return an error tuple, so the accumulator can never become one.
  - `raw: false` → `IO.write/2` calls `:io.put_chars(map_dev(device), …)` (`io.ex:284-286`). I built a **live** io-server that replies `{:io_reply, ReplyAs, {:error, :myfakeerr}}` to `put_chars` and stays alive, then ran `Enum.into(["a\n","b\n"], IO.stream(pid, :line))`. Result: `ArgumentError` raised **inside** `:io.put_chars` (stacktrace top frame `:io.put_chars/2` at `io.erl:205`, `error_info` cause `{:device, :myfakeerr}`), before control ever returns to the collector. So `IO.write` does **not** return `{:error, _}` to Elixir; the accumulator stays `:ok` until the raise. The `FunctionClauseError`-via-accumulator-mismatch path hypothesised in the first pass is therefore **not reachable** even with a cooperating error-replying server. Impact in every case is a raised exception (crash), never silent corruption, injection, bypass, or info leak. Not a security defect. *(This closes the "asserted-narrow" caveat carried in the first pass.)*
- **`IO.StreamError.message/1` interpolation (`stream.ex:10`)** — *Basis: verified.* `inspect(reason)` is inert. `Exception.message(%IO.StreamError{reason: {:enoent, ~c"'; rm -rf /", %{a: 1}}})` produces literal text with no evaluation. `reason` is an internal IO-error term surfaced from `IO.each_stream`/`each_binstream` (`io.ex:829`, `io.ex:843` — `raise IO.StreamError, reason: reason`), not attacker-controlled secret data, so no meaningful info-leak.
- **Public `device` field controllable by caller (`stream.ex:29,43,62`)** — *Basis: reasoned from contract (not a trust boundary).* Pointing `device` at an arbitrary term is a caller contract violation (`device: IO.device()`). The caller — not a remote attacker — chooses the device; `map_dev/1` (`io.ex:855`) accepts atom/pid/tuple and passes anything else through unchanged, and misuse of the documented type spec is explicitly out of scope. No axis to execute here: the value is caller-supplied by definition.
- **Re-enumeration side effects / `Stream.resource` cleanup (`stream.ex:69`)** — *Basis: verified.* The `after` function `& &1` intentionally does not close the caller-owned device; repeated enumeration yielding different data is documented (`stream.ex:24-25`, echoed at `io.ex:644-645`, `io.ex:697-698`). `Stream.resource(fn -> device end, next_fun, & &1)` threads the device as the accumulator; `each_*` returns `{:halt, device}` on `:eof` (`io.ex:825-826`, `io.ex:839-840`). No resource leak or state confusion introduced by this file.

## Risks / caveats

- Scope was this file. Its delegations were resolved by reading the full neighbour `io.ex`: `write` (`284-286`), `binwrite` (`306-310`), `each_stream` (`822-834`), `each_binstream` (`836-848`), `stream`/`binstream` constructors (`666-671`, `707-712`), and `map_dev` (`850-855`). The *runtime* io-server / `:unicode` behaviour behind `:io.put_chars`/`read`/`binread` (e.g. a hostile io-server driving an unbounded `:line` read, or chardata validation) lives in Erlang/OTP below `io.ex` and is outside both this file and its Elixir neighbour; it is not reachable as a defect originating in `stream.ex`.
- No `.git` history was present, so I could not machine-diff against a pinned upstream commit; the behavioural characterizations above are from reading every line of `stream.ex` and `io.ex` and from runtime execution, not a byte-level diff.

## Did I verify this?

- **`IO.binwrite` raises rather than returns an error tuple** — verified by reading `io.ex:306-310`.
- **`IO.write` → `:io.put_chars` raises inside Erlang `io` on a live server's `{:error,_}` reply, never returning an error tuple to the collector** — verified by execution (custom live io-server; observed `ArgumentError` at `:io.put_chars/2`, `io.erl:205`, cause `{:device, :myfakeerr}`; collector never re-entered with a non-`:ok` accumulator). This is the axis reading alone could not settle, and it is now tried.
- **`each_stream`/`each_binstream` convert `{:error, reason}` into a raised `IO.StreamError`, so the Enumerable side never yields an error tuple downstream** — verified by reading `io.ex:822-848`.
- **`line_or_bytes` is guarded to `:line | pos_integer` before reaching this file** — verified by reading the constructor guards `io.ex:667-669` and `io.ex:708-710`.
- **`inspect(reason)` in `IO.StreamError.message/1` is inert** — verified by execution (adversarial `reason` term rendered literally, no evaluation).
- **Re-enumeration does not close the device and is documented** — verified by reading `stream.ex:24-25,69` and `io.ex:644-645,697-698`.

### Dropped in re-review

No prior findings existed to drop (the first pass reported none). The first pass's residual caveat — that the `raw: false` write-error path was a real `FunctionClauseError` whose reachability was "asserted-narrow" — is **withdrawn** as a caveat: executed evidence shows `:io.put_chars` raises `ArgumentError` before the collector can observe an error accumulator, so that specific mismatch is unreachable. The security conclusion (no findings; worst case is a crash) is unchanged.
