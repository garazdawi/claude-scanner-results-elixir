# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/stream.ex
<!-- sha256: 75b39731bd6d0a1c40ab81f160af4e9e1d2f566d37b49faf787cc1e758bbc3b7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T23:44:38Z_

I have confirmed a genuine bug. Let me write the report.

## Summary

`Stream.chunk/3` (the deprecated but still-exported chunking function) passes `nil` as the leftover argument instead of `:discard`, causing a `Protocol.UndefinedError` crash whenever the input does not divide evenly into full chunks.

## Bugfixes

#### `CSCAN-stream-chunk3-nil-leftover`
- **Finding ID:** `CSCAN-stream-chunk3-nil-leftover`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/stream.ex:159-161`, `Stream.chunk/3`
- **Issue:** `def chunk(enum, n, step)` forwards to `chunk_every(enum, n, step, nil)`, passing `nil` as `leftover`. The sibling clauses guard against this: `chunk/2` routes through `chunk/4` and `chunk/4` itself normalizes with `leftover || :discard` (line 167). `chunk/3` skips that normalization, so `nil` reaches `Stream.Reducers.chunk_every/5`, which calls `Enum.take(nil, …)` on the leftover to fill the final partial chunk — and `Enumerable` is not implemented for `nil`.
- **Class:** Missing clause / non-total function (an un-normalized `nil` reaches a path expecting an enumerable or `:discard`).
- **Impact:** Any caller of the documented `Stream.chunk(enum, n, step)` crashes with an opaque `Protocol.UndefinedError` ("protocol Enumerable not implemented for Atom / Got value: nil") whenever the enumerable's length is not a multiple that lets every chunk fill completely — i.e. the common case. The error message points at `Enum.take`/reducer internals, telling the caller nothing about the actual mistake. The historical contract of `chunk/3` is to *discard* the trailing incomplete chunk, so it should silently return the complete chunks. Even-length inputs work, masking the bug in trivial tests.
- **Evidence:** `verified`.
  - `Stream.chunk([1,2,3,4,5], 2, 2) |> Enum.to_list()` →
    ```
    ** (Protocol.UndefinedError) protocol Enumerable not implemented for Atom
    Got value: nil
        (elixir) lib/enum.ex: Enum.take/2
        (elixir) lib/stream/reducers.ex:171: Stream.Reducers.chunk_every/5
        (elixir) lib/stream.ex:305: Stream.after_chunk_while/2
    ```
  - `Stream.chunk([1,2,3,4], 2, 2) |> Enum.to_list()` → `[[1, 2], [3, 4]]` (even split, no crash — confirms the crash is triggered only by an incomplete final chunk).
  - Contrast `Stream.chunk([1,2,3,4,5], 2, 2, nil) |> Enum.to_list()` → `[[1, 2], [3, 4]]` (the `chunk/4` `nil || :discard` normalization saves it), proving `chunk/3` is the only broken arity.
- **Suggested fix:** Change line 160 to `chunk_every(enum, n, step, :discard)` (matching the discard-the-remainder semantics that `chunk/2` and `chunk/4`-with-`nil` already deliver).

## Risks / caveats

- `Stream.chunk/3` is `@deprecated` (`"Use Stream.chunk_every/3 instead"`), but it is still a compiled, exported, documented public function — deprecation emits a warning, not a removal — so callers on the documented path reach the defect. I judged it in scope for that reason; a reviewer who scopes deprecated functions out would downgrade it.
- I ran the file's own doctests indirectly by exercising the sampled examples (`chunk_every`, `map_every`, `cycle`, `with_index`, `from_index`) and they behaved as documented; I did not execute the full doctest suite, so a subtle doctest/timing mismatch (e.g. the `interval/1` blocking example) was not exhaustively checked.
- `from_index/2`'s function clause guards on `is_function(fun)` (any arity) while its `@spec` and body assume 1-arity; passing a 0- or 2-arity function crashes. I excluded it as a `@spec`-contract violation per the scope rules (the caller must supply the documented arity), not a bug in the file.

### Ruled out
- **`interval/1` / `timer/1` `when is_integer(n) and n >= 0 when n == :infinity` guard** — the double-`when` is a guard disjunction (OR), correctly admitting both non-negative integers and `:infinity`; verified `Stream.timer(0) |> Enum.to_list()` returns `[0]` and the guard structure is standard Elixir. Basis: `asserted` (guard semantics) plus the `timer(0)` behavioural check.
- **`do_transform`/`do_resource` catch-and-reraise asymmetry** — every `try` in the transform/resource machinery catches `kind, reason` (all of `:throw`/`:error`/`:exit`), runs `after_fun`/`next.({:halt,[]})`, and reraises with the preserved stacktrace; no original error is swallowed. Searched all `catch` clauses in lines 588–1095 and 1580–1672. Basis: `asserted` (read of every catch clause).
- **`cycle/1` empty-enumerable handling** — the list clause raises `ArgumentError` on `[]` directly, and the general clause raises via `check_cycle_first_element` on a first-pass-empty reduce; the `elem(acc,0) != :halt` guard prevents a spurious raise when the consumer halts. Basis: `asserted` (control-flow read), consistent with the documented "cannot cycle over an empty enumerable".

