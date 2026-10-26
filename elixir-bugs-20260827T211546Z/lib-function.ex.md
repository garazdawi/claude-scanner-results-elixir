# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/function.ex
<!-- sha256: d6d1ee6bf3e5d6e18d7ec94439bb48c563788571a5bb7b5330d91b5124e17f97 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/function.ex
<!-- sha256: d6d1ee6bf3e5d6e18d7ec94439bb48c563788571a5bb7b5330d91b5124e17f97 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28 (hedge-resolution pass over the 2026-08-27 first pass; flagged hedge: "outside this file")_

## Summary
`Function` is a thin wrapper over `:erlang.make_fun/3` and `:erlang.fun_info/1,2` plus `identity/1`. The flagged hedge ("the real behavior of `make_fun/3` and `fun_info/1,2` lives in OTP … outside this file's scope") is the crux of this file's only finding, so I discharged it the only valid way — **by execution**, not by reading a neighbour. The Phase-instruction is explicit that reading a module tells you what a guard does, not which inputs disagree; here the relevant "neighbour" is OTP's `:erlang` BIF, which is not a readable source module in this tree, so execution is the sole verification path. I re-ran every delegated BIF on OTP 29.

Result: pushing a *local anonymous function* through `:erlang.fun_info/1` returns a `{:pid, pid()}` pair whose key `:pid` is **not** a member of the module's `@type information` union and is **not** documented by `info/1`. That makes the `@spec` on `info/1`/`info/2` inaccurate against real runtime output. One low-severity finding stands, executed and reproduced. The first pass's "spec enumerates exactly the keys `fun_info` returns" ruling remains dropped (falsified by execution).

- **Checkout caveat:** the target `function.ex` is absent from this environment (path `.../lib/elixir/lib/function.ex` does not exist; only `.../lib/elixir/src/*.erl` is present). Line-number anchors below are inherited from the prior report and were not re-confirmed against source text. All behavioural claims were re-executed.

## Bugfixes

### 1. `@type information` / `info/1` doc omit the `:pid` key that `:erlang.fun_info/1` actually returns — `@spec` inaccurate
- **Location:** `function.ex:64-73` (`@type information`), `function.ex:92-147` (`info/1` doc + `@spec info(fun) :: [{information, term}]`), `function.ex:182` (`@spec info(fun, item) :: {item, term} when item: information`). *(Anchors inherited from prior report; source not re-readable in this checkout.)*
- **Sink class:** documented-contract / return-shape violation (`@spec`/`@type` accuracy — the OTP-internal "documented return shape" contract, the `fun_info` analogue of DOCUMENTED_TOKENIZER_RETURN_SHAPE, claim 16).
- **What I ran (OTP 29, Elixir 1.21.0-dev, re-executed this pass):**
  ```
  fun = fn x -> x end
  Keyword.keys(Function.info(fun)) |> Enum.sort()
  # => [:arity, :env, :index, :module, :name, :new_index, :new_uniq, :pid, :type, :uniq]
  :erlang.fun_info(fun, :pid)                # => {:pid, #PID<0.0.0>}
  Function.info(fun, :pid)                   # => {:pid, #PID<0.0.0>}
  :erlang.fun_info(&String.length/1, :pid)   # => {:pid, :undefined}
  Keyword.keys(Function.info(&String.length/1)) |> Enum.sort()
  # => [:arity, :env, :module, :name, :type]
  ```
- **Defect:** `info/1` on a local anonymous function returns a list that includes `{:pid, pid()}`, but `:pid` is absent from the `information` union (`function.ex:64-73`). Therefore:
  - `@spec info(fun) :: [{information, term}]` (`function.ex:146`) is violated by the BIF's own output — a Dialyzer-checked consumer folding the list against `information` sees an out-of-contract key.
  - `@spec info(fun, item) :: {item, term} when item: information` (`function.ex:182`) rejects the legal call `Function.info(fun, :pid)`, which in fact returns `{:pid, pid()}`.
  - The `info/1` doc (`function.ex:98-122`) enumerates the returned keys explicitly and omits `:pid`, so the documentation is incomplete for local anonymous functions.
- **Failure scenario:** a caller writes `Function.info(fun, :pid)` (or Dialyzer-checks a fold over `Function.info(fun)`); it works at runtime but fails the published contract / triggers a Dialyzer `call` warning, because `:pid` is not in `information`.
- **Fix:** add `| :pid` to the `information` union and document `:pid` under the anonymous-function key list in `info/1`.
- **Confidence:** Medium. The runtime discrepancy is executed and reproducible (the fact itself is High-confidence); rated Medium overall because severity is low — `info` is documented "for debugging purposes only" and the mismatch is a type/doc-completeness defect, not a crash or miscompute. (I could not re-read the `@type information` source text to 100% confirm `:pid` is absent from the union rather than present-but-undocumented; the first pass reported it absent, and the doc omission is independently confirmed by the OTP `fun_info` docs. This residual is why the finding is not raised to High.)

### Ruled out
- **`capture/3` doctest** (`Function.capture(String, :length, 1)` ⇒ `&String.length/1`) — re-ran this pass; `Function.capture(String, :length, 1) == (&String.length/1)` is `true` and `inspect/1` yields `"&String.length/1"`. Basis: `verified`. Axes examined: value-equality, inspect form | Not examined: none.
- **`info/1` and `info/2` doctests** — re-ran this pass: `local arity={:arity,1}`, `local type={:type,:local}`, `ext name={:name,:length}`, `ext type={:type,:external}`; all match. Basis: `verified`. Axes examined: local fn, external capture, both arities | Not examined: none. (The *set* of keys returned by `info/1` is NOT ruled out — see finding 1.)
- **`identity/1` doctests** — re-ran `chunk_by` on sorted `~c"abcdaabccc"` ⇒ `[~c"aaa", ~c"bb", ~c"cccc", ~c"d"]` this pass; matches. (Prior pass also confirmed the string, `group_by`, and `Enum.map` doctests.) Basis: `verified`. Axes examined: charlist, string, integer list | Not examined: none.
- **`identity/1` Unicode / malformed-term preservation** — re-executed this pass: `Function.identity(<<0xFF,0xFE,0x00>>) === <<0xFF,0xFE,0x00>>`, `Function.identity([1|2]) === [1|2]`, and `Function.identity(make_ref()) === r` each returned `true`. Identity holds under strict `===` for non-UTF-8-binary, improper-list, and reference axes. Basis: `verified`. Axes examined: bad binary, improper list, reference, charlist, string | Not examined: none.
- **`@spec capture / identity` accuracy** — `capture(module, atom, arity) :: fun` matches the executed `make_fun/3`/`capture/3` result (a `fun`, `inspect` `&String.length/1`); `identity(value) :: value when value: var` matches the strict-identity behaviour re-confirmed above. Basis: `verified`. Axes examined: return shape vs run output | Not examined: none.

## Risks / caveats
- The flagged hedge — that `make_fun/3` / `fun_info/1,2` behaviour "lives in OTP … outside this file's scope" — is now **discharged by execution against OTP 29**, not assumed. The relevant behaviour is in a compiled BIF, not a readable neighbour source module, so execution (not a `Read`) is the correct and only verification; that is what I did.
- The target source file is not present in this checkout, so I could not re-verify the exact `@type information` union text or the `info/1` doc key list against source. Finding 1's runtime basis is fully executed; its *source-side* premise (`:pid` absent from the union) is carried from the first pass and corroborated by the OTP docs, and is the reason the finding is held at Medium rather than High.
- The sandbox VM emitted latin1-locale warnings; these did not affect any result (all doctest data is ASCII; the term-preservation probes use raw bytes/refs, unaffected by name encoding).

## Did I verify this?
Yes, by execution (Erlang/OTP 29, Elixir 1.21.0-dev), re-run this pass:
- Ran all doctest expressions across `capture/3`, `info/1`, `info/2`, `identity/1` — all match.
- Ran `Keyword.keys(Function.info(fn x -> x end))`, `:erlang.fun_info(fun, :pid)`, `Function.info(fun, :pid)`, and the external-capture equivalents — established `:pid` is returned as `{:pid, pid()}` for local funs and `{:pid, :undefined}` for external captures, and that external captures return only `[:arity, :env, :module, :name, :type]`. Basis for finding 1.
- Ran `identity` over a malformed binary, an improper list, and a reference under `===` — all identical.
- Could NOT re-read `function.ex` source (absent from this checkout); source-side line/type-text claims are inherited, not re-verified — stated as a caveat, not hidden.

## Dropped in re-review
- **First-pass ruled-out entry "`@spec` accuracy … `information` type enumerates exactly the keys `:erlang.fun_info` returns … Basis: `verified`".** Remains dropped: falsified by execution. `Function.info(fn x -> x end)` returns a `:pid` key that the `information` union does not contain. The accurate portions of that entry (return *shapes* of `capture`, `info/2` tuple form, `identity`) are retained above as separate `verified` ruled-out entries; the incorrect "enumerates exactly" sub-claim is superseded by finding 1. No other prior finding is dropped.
