# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/list/chars.ex
<!-- sha256: 41c0911fe6a5c9c69b88c1e97ae189b99315b2631542ece5ccd8c93777d09f39 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:35:27Z_

I now have a complete, verified picture. Writing the report.

---

## 1. Summary

`list/chars.ex` defines the `List.Chars` protocol and five thin, faithful type-conversion implementations (`Atom`, `BitString`, `List`, `Integer`, `Float`); it contains no dangerous sinks and no reachable security defect.

## 2. Security

**No findings.**

This file's only documented public API is `List.Chars.to_charlist/1` (invoked by the `Kernel.to_charlist/1` macro, `kernel.ex:3512`). Every implementation is a faithful, well-defined conversion to a charlist. There is no code execution, atom creation, command/file/network I/O, deserialization, or shared-state write anywhere in the file. Charlist interpolation (`~c"#{…}"`) does **not** route through this protocol — it uses `List.to_charlist/1`/`:unicode.characters_to_list` (`kernel.ex:6622`, `list.ex:1290`) — so the interpolation-injection surface that normally makes the `*.Chars` protocols interesting does not touch this file.

Each Phase-1 sink class was traced and settled by running the built-from-source toolchain (`/opt/elixir`, 1.21.0-dev):

### Ruled out

- **Atom creation (CWE-770).** `Atom.to_charlist/1` (`chars.ex:27`) *consumes* an already-interned atom and emits a charlist; it cannot mint atoms. `verified`: 300 000 `to_charlist/1` calls over pre-created atoms produced an `atom_count` delta of **0**. The `nil → ~c""` special-case (`chars.ex:25`) is a constant. No untrusted-bytes-to-atom path exists.

- **List passthrough returns non-charlists unchanged** (`chars.ex:48`, `def to_charlist(list), do: list`). `verified`: `List.Chars.to_charlist([:a,:b,:c])` → `[:a, :b, :c]` and `List.Chars.to_charlist([0x110000, -1])` → `[1114112, -1]`, i.e. the `@spec … :: charlist` output contract is not enforced. This is the intentional, documented performance inlining ("same inlining is used for the rewrite rule"), and it is *not* a security issue in this file: the function introduces no new data and no escaping/validation was ever promised for the List branch. Any hazard exists only if a *caller* feeds the unchanged list to a downstream sink that assumes a valid charlist — that danger lives at the sink, not here, and would require the caller to treat an unvalidated list as validated. Worth a one-line caveat below, not a finding.

- **Template/interpolation escaping.** The conversions perform no escaping (`String.to_charlist("a; rm -rf")` yields the literal chars). Correct by design: `List.Chars` is a representation protocol, not a sanitizer; escaping is the responsibility of the eventual interpreter (shell/query/markup), not the conversion. No finding.

- **Malformed-UTF-8 / conversion errors surfaced.** `BitString.to_charlist/1` delegates binaries to `String.to_charlist/1`. `verified`: `to_charlist(<<0xFF,0xFE>>)` raises `UnicodeConversionError` rather than emitting replacement/garbage — the decode-error-surfaced obligation is met by delegation. No silent corruption.

- **Dead / unreachable defensive code.** The `BitString` catch-all (`chars.ex:38-43`) is reachable and correct: `verified` that a sub-byte bitstring `<<1::4>>` raises `Protocol.UndefinedError` (it is a bitstring but not a binary, so it falls past the `is_binary/1` guard). The guard fires; it is not dead.

- **Resource consumption (DoS).** `String.to_charlist`, `Integer.to_charlist`, `Float.to_charlist` are all linear in input size with no super-linear amplification; the output is proportional to an input the caller already holds in memory. No quadratic blowup or unbounded loop. No finding.

- **Unicode identifier-security obligations (bidi / NFC / mixed-script / confusables / escape validation).** Not applicable to this file — those checks live in `elixir_tokenizer.erl` / `elixir_interpolation.erl`, which handle source *identifiers* and *escapes*. This file only performs runtime value-to-charlist conversion and implements none of that machinery, so there is nothing here to weaken or bypass.

## 3. Risks / caveats

- **Search space:** I reviewed only `list/chars.ex` and traced its public entry point (`Kernel.to_charlist/1`) and the charlist-interpolation path into adjacent files (`kernel.ex`, `list.ex`) enough to confirm this protocol is *not* on the interpolation path. Findings in `String.to_charlist`, `Integer.to_charlist`, `List.to_charlist`, or the tokenizer are out of scope and not assessed here.
- **Non-security correctness note (not a finding):** the `List` implementation trusts its input is already a valid charlist and returns it verbatim, so `to_charlist/1` must not be relied upon as a *validator* of list contents — it will hand back atoms, negative integers, or out-of-range codepoints unchanged. Consumers that later pass the result to a charlist-strict sink (e.g. `:os.cmd/1`, `IO.chardata_to_string/1`, `:erlang.list_to_binary/1`) should validate independently. This matches documented protocol behavior and is a caller expectation, not a defect in this file.
- **Reproductions** were run against `/opt/elixir` (Elixir 1.21.0-dev / OTP 29), which reports as built from this source tree; behavior of a differently-compiled build was not checked, but these are stable protocol semantics.

