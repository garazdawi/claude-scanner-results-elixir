# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/io/stream.ex
<!-- sha256: 8395113d478990bf6f24ced4c14ef9b611cb9e198ef68112e84f73c7091b0cfb -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:22:27Z_

## Summary

`IO.Stream` is a small, verbatim-standard struct with `Collectable` and `Enumerable` implementations; I found no genuine reachable bugs — the one theoretically suspicious path (error accumulation in the collector) is unreachable because `IO.write`/`IO.binwrite` raise rather than return `{:error, _}`.

## Bugfixes

No findings

### Ruled out

- **Collector accumulator crashes on write error** (`stream.ex:47-58`, `Collectable.into`). The collector fun threads the return value of `IO.binwrite`/`IO.write` as the next accumulator and only has heads for `:ok, {:cont, x}` / `:ok, _`. If a write returned `{:error, reason}`, the next `{:cont, _}` would hit no clause (`FunctionClauseError`). But in current OTP this is unreachable: `IO.write` calls `:io.put_chars`, which *raises* `ArgumentError` on a device error (verified: `IO.write(dev, "x")` against a device replying `{:error, :myerr}` raised `ArgumentError ... unknown error: myerr`, not a returned tuple), and `IO.binwrite` (`io.ex:306-310`) explicitly does `with {:error, reason} <- :file.write(...), do: :erlang.error(reason)`. So the collector only ever gets `:ok` back or an exception propagates — the accumulator never becomes an error tuple. **Basis: verified.**
- **Error propagation on a dead/closed device** — writing into a stream whose device pid is dead raises `ErlangError :terminated` from the write itself (verified via `Enum.into(["a\n"], IO.stream(closed_pid, :line))`), before any accumulator threading, so no malformed-accumulator state arises. **Basis: verified.**
- **`Collectable.into/1` return shape** returns `{:ok, collector_fun}`; initial acc `:ok` matches the collector's `:ok, _` heads, and the `:done`/`:halt` branch returns the `stream` struct (the collection), per the `Collectable` contract. Round-trip verified: `Enum.into(["a\n","b\n","c\n"], IO.stream(pid,:line))` produced `"a\nb\nc\n"`; raw `IO.binstream(pid,2)` produced `"xyzw"`. **Basis: verified.**
- **`Enumerable` `reduce`/`count`/`member?`/`slice`** — `reduce` delegates to `Stream.resource/3` over the device (verified read-back: `Enum.take(IO.stream(pid,:line),5)` → `["a\n","b\n","c\n"]`); the other three correctly return `{:error, __MODULE__}` (the impl module) to signal non-enumerable-count, the standard convention. **Basis: verified.**
- **`defstruct`/`@type`/`__build__`** defaults (`device: nil, raw: true, line_or_bytes: :line`) and `IO.StreamError.message/1` are consistent with the `@type t` and the documented public fields; no mismatch. **Basis: asserted (pure structural inspection of the whole file — the entire module is the 85 lines read).**

## Risks / caveats

- The file is a faithful copy of the upstream `IO.Stream`; the collector's reliance on writes returning exactly `:ok` is an implicit coupling to `IO.write`/`IO.binwrite`'s raise-on-error behavior in `io.ex`. It is currently safe, but a future change making those functions return `{:error, _}` (matching the older `@spec :: :ok | {:error, term}`) would reintroduce the latent `FunctionClauseError` in the collector. Worth a defensive `_, {:cont, _}` head if that contract ever loosens.
- I did not exercise every exotic IO device (e.g. custom `:io`-protocol servers with partial request support); testing covered file devices (read/write/closed), a raw error-returning device, and normal round-trips, which are the paths reachable from the documented `IO.stream/2`/`IO.binstream/2` entry points.

