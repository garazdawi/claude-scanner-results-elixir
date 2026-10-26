# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel.ex
<!-- sha256: 33d922b91dbb838c9a8f4d06b0dd73fdadabe0126035846aa62704d1fc9c79ec -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:29:07Z_

All four agents are complete. Consolidated result:

- **Agent 1 (400–2200):** two verified failing doctests — `or/2` (line 2028) and `and/2` (line 2059), stale single-line `BadBooleanError` message.
- **Agent 2 (2200–4200):** no bugs; all doctests pass.
- **Agent 3 (4200–5700):** no bugs; file byte-identical to upstream `main`.
- **Agent 4 (5700–7113):** no bugs; file byte-identical to upstream `main`.
- **My manual pass** (get_in/put_in/struct family, if/unless/&&/||, in/2, ranges, raise/reraise, match?, is_struct, `**`, binary_slice, defmodule/def, `@`): all correct.

I independently reproduced the one finding (2/6 doctests fail against the locally-built compiler). Here is the report.

---

## Summary

The file is byte-for-byte identical to upstream Elixir `main` and is almost entirely correct; the only reachable defect found is a pair of stale doctests for `and/2` and `or/2` whose expected `BadBooleanError` message no longer matches the runtime's now-multi-line format, so they fail when run.

## Bugfixes

#### `CSCAN-kernel-and-or-badboolean-doctest`
- **Finding ID** — `CSCAN-kernel-and-or-badboolean-doctest`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/elixir/lib/kernel.ex:2028` (`defmacro left or right`) and `lib/elixir/lib/kernel.ex:2059` (`defmacro left and right`)
- **Issue** — Both `@doc` blocks contain a doctest asserting the raised `BadBooleanError` message on a single line: `** (BadBooleanError) expected a boolean on left-side of "or", got: 42` (line 2028) and `... got: "yay!"` (line 2059). At runtime, `BadBooleanError.message/1` now delegates to `Exception.__format_message_with_term__`, which renders the offending term on its own indented line, producing `expected a boolean on left-side of "or", got:\n\n    42\n`. The documented (doctest) side is the stale one; the multi-line formatter is intentional (documented in `exception.ex`).
- **Class** — Documentation mismatch (failing doctest).
- **Impact** — Anyone running `doctest Kernel` (or copying the documented example expecting that exact message) sees a failure/mismatch. The operators themselves behave correctly — only the documented error-message text is wrong — so functional impact is nil; the cost is a red doctest and misleading docs.
- **Evidence** — `verified`.
  - Actual message: `elixir -e 'try do Code.eval_string("42 or false") rescue e -> IO.inspect(Exception.message(e)) end'` →
    `"expected a boolean on left-side of \"or\", got:\n\n    42\n"`
  - Doctest run via ExUnit (`doctest Kernel, only: [or: 2, and: 2]`) →
    ```
    Result: 4/6 passed
    Failed: 2 doctests
    1) doctest Kernel.or/2 ... expected: "... got: 42"  actual: "... got:\n\n    42\n"  (kernel.ex:2027)
    2) doctest Kernel.and/2 ... expected: "... got: \"yay!\""  actual: "... got:\n\n    \"yay!\"\n"  (kernel.ex:2058)
    ```
- **Suggested fix** — Update both doctests to the current multi-line rendering, e.g.:
  ```
  iex> 42 or false
  ** (BadBooleanError) expected a boolean on left-side of "or", got:
  
      42
  ```
  (and likewise for `and/2` with `"yay!"`), or convert them to non-`iex>` (documentation-only) examples if the multi-line form is awkward to embed.

## Risks / caveats

- **Coverage.** The 7113-line file was swept by four independent agents over disjoint ranges (400–2200, 2200–4200, 4200–5700, 5700–7113), each verifying doctests/`@spec`/behaviour by running the locally-built `elixir 1.21.0-dev`, plus a manual pass over the highest-risk pure-logic and macro sites (get_in/put_in/update_in/get_and_update_in/pop_in, struct/struct!, if/unless/&&/‖, in/2, ranges, raise/reraise, match?, is_struct, `**`, binary_slice/2,3, to_timeout, defmodule/def, `@`). Lines 1–400 are moduledoc prose plus the first thin `:erlang` wrappers and were read directly.
- **Upstream parity.** Three of the four agents confirmed (via `diff`/`md5sum`) that this file is byte-identical to upstream `elixir-lang/elixir` `main`. This means no injected/planted defect exists in the file; any finding must be a latent issue in shipping Elixir itself. The doctest finding above is such a latent issue, reproduced against the exact source.
- **Uncertainty on the finding's blast radius.** I did not determine whether Elixir's own CI actually executes `doctest Kernel` for these two entries (if it does, this would already be caught; that it ships this way suggests these specific doctests may be excluded or the multi-line format is a recent change). Regardless, per the review contract "a doctest that cannot currently pass is a finding," and I verified it cannot pass against the built compiler. Severity is Low precisely because the operators work correctly and only the documented message text is stale.
- Nothing else met the bar. Candidate edge cases explicitly checked and cleared include: `in_range_literal/4` not re-checking `is_integer` on bounds (safe — `is_integer(left)` + term ordering suffice), the function-key vs value-function clause ordering in `get_and_update_in/3`/`pop_in_data`, the `is_map_key`/`binary_part` argument order into `:erlang`, `integer_pow` (brute-forced), and `binary_slice/2` step comprehension on non-empty parts.

### Ruled out
- **`and/2`/`or/2`/`&&`/`‖` truthiness and short-circuit semantics** — return values (`false or 42 → 42`, `true and "yay!" → "yay!"`, `false && throw(:bad) → false`) all correct; only the *error-message* doctests fail. Basis: `verified` (ExUnit doctest run, 4/6 passing were exactly these value cases).
- **`get_in`/`put_in`/`update_in`/`get_and_update_in`/`pop_in` (fn + list forms and macro forms), `struct`/`struct!`** — all doctests including every `nil`-raise path pass; clause ordering and `:erlang.element(2, …)` extraction correct. Basis: `verified` (agent 2 ran each; I read the source).
- **`**`/`integer_pow`** — brute-forced `b∈-5..5, e∈0..20` against `:math.pow`, zero mismatches; `0**0=1`. Basis: `verified`.
- **`in/2` guard/body, ranges, `binary_slice/2,3`, `to_timeout/1`** — all doctests plus extra step/negative-bound cases reproduced exactly. Basis: `verified`.
- **All `sigil_*`, `defstruct`/`defexception`/`defprotocol`/`defoverridable`/`defdelegate`/`defguard`/`use`/`dbg`** — doctests and embedded examples verified; logic matches upstream. Basis: `verified` (agent 4 executed the 33 doctests in range).

