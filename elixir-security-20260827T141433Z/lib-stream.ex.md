# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/stream.ex
<!-- sha256: 75b39731bd6d0a1c40ab81f160af4e9e1d2f566d37b49faf787cc1e758bbc3b7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:30:46Z_

The grep confirms my full read: the only executable sink in the module is `Process.sleep(n)` at line 572 (everything else is doc-comment/doctest text). Every other line is data-transformation over caller-supplied enumerables and functions.

Here is the report.

---

## Summary

`Stream` is pure, lazy data-transformation code whose only inputs are a caller-supplied enumerable and caller-supplied callback functions; no element data reaches a code-execution, atom, command, path, deserialization, crypto, or template sink, so there are no security findings reachable through its documented public API.

## Security

`No findings`

### Ruled out

- **`Process.sleep(n)` in `interval/1` (line 572) / `timer/1` (line 855) — sleep-based DoS.** `n` is a *direct argument* to the public function, guard-constrained to `is_integer(n) and n >= 0` or `:infinity`, and matches the documented `@spec interval(timer())`. Blocking the caller "by the given interval every time a new element is streamed" is the function's stated purpose (lines 557–560, 842–843). A caller who forwards an attacker-controlled duration into a sleep primitive owns that decision; the value is not element data flowing through a stream, and there is no amplification. *Basis: verified — grep of the whole file confirms this is the sole `Process.sleep`/blocking call site; behavior is documented in-source.*
- **`uniq/1`, `uniq_by/2` unbounded memory (lines 1122, 1154).** Stores every unique value in a map. This is explicitly documented as expected ("needs to store all unique values… if the stream is infinite, the number of elements stored will grow infinitely," lines 1110–1113, 1139–1142). Cost is linear in the input the caller *chose* to enumerate — no small-input-to-huge-cost amplification distinct from `Enum` on the same data. *Basis: verified by read; behavior documented in source.*
- **Negative-count buffering in `take/2` (`-count`, lines 784–786) and `drop/2` (`-n`, lines 368–392).** Both buffer a bounded window (`2 * count` for `take`, `n` for `drop`) where the window size is a *caller-supplied integer argument*, not element data, and both are documented (lines 750–754, 346–350). No data-driven blowup. *Basis: verified by read; documented in source.*
- **No code-execution / atom-creation / command / path / archive / deserialization / crypto / template / regex sinks.** A grep across the entire file for `to_atom`, `Module.concat`, `binary_to_term`, `Code.*`, `System.cmd/shell`, `:os.cmd`, `Regex.compile`/`~r`, `Path.*`, `File.*`, `:erl_tar`/`:zip`, `EEx`, `Base.decode`, `URI.*`, `:crypto`, `persistent_term`/`:ets`/`Application.put_env`/`Process.put`, `apply`, `spawn`, `send` returns only doc-comment/doctest text (lines 95, 104, 485, 697–699, 1547–1554) plus the `Process.sleep` above. None of the dangerous named functions are called by module code. *Basis: verified — full 1823-line read plus the grep above define the search space.*
- **Protocol dispatch (`Enumerable.reduce`, `Collectable.into`, `Enumerable.List.reduce`).** Dispatch is on the type of the caller-provided enumerable/collectable — the documented contract of every `Stream` function — not on an externally-derived module-name atom, and no `defimpl`/`@derive` is selected from element data. Element values are only ever handed to caller-supplied callbacks or accumulated into lists/maps/collectables. *Basis: verified by read.*
- **Private macros `next`, `next_with_acc`, `acc`, `skip` (lines 132–149) using `unquote`.** They splice a fixed internal AST shape, never caller-supplied expressions, and run only at compile time of this module with no filesystem/network access — no unquote-hygiene or build-time-execution exposure. *Basis: verified by read.*
- **List recursion in `zip_pair/5` (lines 1343–1356).** The recursive clause is in tail position (arguments evaluated, then the tail call), so no stack growth on long caller lists. *Basis: verified by read.*

## Risks / caveats

- Scope was this file only. `Stream` composes with `Enum`, `Enumerable`/`Collectable` protocol implementations, and `Stream.Reducers` (the `R.*` macros); a defect in those modules is out of scope here and was not reviewed. The `R.*` reducer macros expand into this module but their bodies live in `Stream.Reducers`.
- The whole-module claim of "no dangerous sinks" rests on a complete read plus the grep above; it is a claim about *this file's* call sites, not about what a caller's own callback functions do. Any `map`/`filter`/`reduce`/`transform`/`resource` callback the caller passes runs with full VM privileges — that is the documented, intended contract of a higher-order lazy-enumeration library and not a defect in `Stream`.
- Documented resource-consumption behaviors (`uniq` retention, negative `take`/`drop` buffering, `cycle`/`iterate`/`repeatedly` infiniteness, `interval`/`timer` blocking) are real availability considerations for a caller who feeds them untrusted data or durations, but each is bounded by a caller-chosen argument or the caller-chosen input size and is explicitly documented, so none rises to a library vulnerability.

