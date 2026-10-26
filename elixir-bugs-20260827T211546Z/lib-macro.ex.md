# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/macro.ex
<!-- sha256: 30d130be4957f46ab72c0524afa2b73c2237b66985c2555b001b0fbdfeab27bf -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:41:15Z_

# Bug-hunting review: `lib/elixir/lib/macro.ex`

## Summary
One genuine defect: the `Macro.pipe/3` clause for `Access.get` calls captures the wrong sub-term (`op = [Access, :get]`) and, on its non-bracket branch, emits a structurally invalid AST that fails to compile; the rest of the module's public API round-trips and doctests (46/46) pass.

## Bugfixes

#### `CSCAN-macro-pipe-access-get-malformed-ast`
- **Finding ID:** `CSCAN-macro-pipe-access-get-malformed-ast`
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/elixir/lib/macro.ex:317-337`, `def pipe/3` (the `Access.get` clause), specifically the `else` at line 335.
- **Issue:** The pattern binds `[Access, :get] = op`, so `op` is the argument **list** of the `.` node, not the `.` node itself. On the non-bracket branch the call is rebuilt as `{op, meta, List.insert_at([first, second], integer, expr)}`, i.e. `{[Access, :get], meta, [...]}` — a 3-tuple whose first element is a list, which is not a valid AST call node. It should reconstruct `{{:., dot_meta, [Access, :get]}, meta, [...]}` (i.e. capture the whole `{:., _, [Access, :get]}` node), producing `Access.get(expr, first, second)`.
- **Class:** Pattern match on the wrong shape (capture at wrong nesting level) → produces malformed AST.
- **Impact:** A caller who pipes into an already-alias-expanded `Access.get/2` call node (literal atom `Access`, two args, no `:from_brackets` metadata — the shape produced after `Macro.expand`/programmatic AST construction) gets back a broken AST instead of `Access.get(expr, a, b)`. Compilation later fails with the opaque `invalid call [Access, :get](...)` rather than either working or raising a clear `ArgumentError`. Note the asymmetry: piping into the ordinary source form `Access.get(a, b)` (which quotes as `{:__aliases__, …}`) works and yields `Access.get(x, a, b)`; only the expanded-atom form breaks. The `|>` operator itself never hits this branch (bracket syntax always carries `:from_brackets`, taking the raising branch), so exposure is via direct `Macro.pipe/3` use in metaprogramming — hence Medium.
- **Evidence:** `verified`.
  - `Macro.pipe(quote(do: x), quote(do: Access.get(a, b)), 0)` → `Access.get(x, a, b)` (alias form, correct).
  - `Macro.pipe(quote(do: x), {{:., [], [Access, :get]}, [], [a, b]}, 0)` → `{[Access, :get], [], [{:x,…}, {:a,…}, {:b,…}]}` (expanded-atom form, malformed).
  - Feeding that result to `Code.eval_quoted/2` → `error: invalid call [Access, :get](x, a, b)` … `CompileError`.
  - The bracket form `a[b]` still correctly raises `wrong operator precedence when piping into bracket-based access` (the `if` branch is unaffected because `:from_brackets` is present on the dot meta).
- **Suggested fix:** Capture the full dot node instead of its arg list, e.g.:
  ```elixir
  def pipe(expr, {{_, meta, [Access, :get]} = dot, outer_meta, [first, second]}, integer) do
    if {:from_brackets, true} in meta do
      raise ArgumentError, """..."""
    else
      {dot, outer_meta, List.insert_at([first, second], integer, expr)}
    end
  end
  ```

## Risks / caveats
- I read the entire file (3187 lines) and exercised the public surface by running against the locally built Elixir 1.21.0-dev: all 46 `Macro` doctests pass; `to_string/1`, the deprecated `to_string/2`, `unpipe`, `path`, `decompose_call`, `underscore`, `camelize`, `validate`, and the walkers all round-trip on a spread of inputs without crashing or producing wrong output.
- The large hand-rolled `to_string/2` engine (lines 1360–1907) is deprecated but still exported; I fuzzed ~35 constructs through it without finding a crash, but it is combinatorially large and I did not enumerate every operator/precedence/sigil-delimiter combination, so a narrow formatting defect there cannot be fully excluded.
- The `dbg_*` compile-time helpers were reviewed statically (they require an active `Macro.Env` and `Kernel.dbg` wiring to exercise end-to-end); I found no defect but did not run them under a live `dbg/2` expansion.

### Ruled out
- **`decompose_call` doctest `[:Elixir, :M]` vs runtime `[Elixir, :M]`** — not a bug. `inspect(:Elixir)` prints `Elixir` (no colon), but Elixir doctests evaluate the expected block as an expression rather than doing string comparison, so `[:Elixir, :M]` and `[Elixir, :M]` are equal terms; the `decompose_call` doctests pass (Basis: `verified` — ran `doctest Macro, only: [decompose_call: 1]`, 6 passed).
- **`generate_arguments/3` using `String.to_unsafe_atom`** — not a bug; the function exists and `Macro.generate_arguments(2, Foo)`/`(0, Foo)` return the documented shapes (Basis: `verified`).
- **`postwalker/1` using `make_ref()` as an in-band marker `{ref, node}`** — safe; the marker clause matches only tuples whose first element is the bound fresh reference, which cannot collide with AST data; `postwalker` doctest passes (Basis: `verified`).
- **`find_invalid/1` for captured functions** returning `{:error, fun}` — correct; only non-external or closure-carrying funs are rejected, external captures return `nil` (valid), matching the `output` type (Basis: `asserted` from code, consistent with `validate` doctests passing).
- **`pipe/3` unary/operator/`fn`/`&`/`__aliases__` guard branches** — each raises a clear `ArgumentError` as documented; only the `Access.get` `else` branch is defective (Basis: `verified` for the bracket branch and several raise branches).

