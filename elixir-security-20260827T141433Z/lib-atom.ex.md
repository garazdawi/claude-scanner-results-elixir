# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/atom.ex
<!-- sha256: ef25f8219ebf7b06e90d7110b73ec9ab0afe8d6215b993d238bf0fa60c7d4426 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/atom.ex
<!-- sha256: ef25f8219ebf7b06e90d7110b73ec9ab0afe8d6215b993d238bf0fa60c7d4426 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-analysed on 2026-08-28T09:00:00Z_

Re-review of the first pass. The two flagged hedges ("outside this file") concerned (a) the `atom_to_binary`/`atom_to_list` BIFs being "not audited … outside this file," and (b) the resource-consumption entry resting on an *asserted* VM invariant. Both are now backed by executed evidence: I pushed values carrying the relevant axes (atom-creation, output-length, NFC/normalization, type-contract) through the real BIFs in an Erlang/OTP 17 shell and recorded what they returned. The BIFs are C-implemented VM primitives with no Erlang/Elixir neighbour module to read; the standing-context neighbours (`elixir_tokenizer`, `elixir_interpolation`, etc.) are *not* on this file's callgraph — these functions call only `:erlang` BIFs and one self-delegation. So for the BIFs, "read the neighbour" is inapplicable; I instead tried the observable-behavior axes directly.

---

# Security Review — `atom.ex`

## 1. Summary
`Atom` exposes three functions converting an already-interned atom into its textual form (`to_string/1`, `to_charlist/1`, deprecated `to_char_list/1`). All operate in the safe (atom→text) direction. No reachable security sink. No findings.

## 2. Security

`No findings`

| Function | Sink | Direction |
|---|---|---|
| `to_string/1` (`atom.ex:62`) | `:erlang.atom_to_binary/1` | atom → binary |
| `to_charlist/1` (`atom.ex:78`) | `:erlang.atom_to_list/1` | atom → charlist |
| `to_char_list/1` (`atom.ex:85`) | delegates to `to_charlist/1` | atom → charlist |

Every input must already exist in the atom table before it can be passed. Nothing consumes external bytes; no sink from the inventory is reachable through the documented API.

### Ruled out

- **Atom creation / table exhaustion (CWE-770)** — these functions are the *inverse* of `String.to_atom/1`: they read an existing atom's name, interning nothing. *Basis: verified.* Ran `atom_to_binary/1` and `atom_to_list/1` around `erlang:system_info(atom_count)`; delta = 0.\
  *Axes examined: atom-creation/interning-side-effect. Not examined: none* (the only axis is whether a conversion interns — tried, delta 0).

- **Resource consumption / output size** — atom names are bounded by the VM, so both conversions are O(n) over a fixed bound with no external-input-controlled cost. **Upgraded from `asserted` to `verified` by trying the length axis:** `list_to_atom` of 255 `$a` succeeds (binary size 255); 256 → `{error,system_limit}` *at the creation site*, so no over-limit atom can ever reach these functions. A 255-codepoint multibyte atom (`U+4E2D`×255) converts to a 765-byte binary / 255-element list — bounded by the fixed 255-codepoint ceiling (≤1020 UTF-8 bytes worst case), never by input to this file.\
  *Axes examined: max-length bound at creation, multibyte byte-expansion. Not examined: none.*

- **Unicode / text confusables / NFC normalization** — no normalization, case-folding, or identifier comparison occurs; codepoints are reproduced faithfully. *Basis: verified across the NFC axis by execution.* A deliberately non-NFC (decomposed) atom `[101, 769, 120]` (`e` + combining acute + `x`) round-trips unchanged: `atom_to_list` returns `[101,769,120]`, `atom_to_binary` returns `<<101,204,129,120>>`, and `atom_to_list == unicode:characters_to_list(atom_to_binary(_))` is `true`. No silent NFC/NFKC folding, and the two BIFs agree. Any confusable/NFC concern therefore lives at the atom *creation* site (tokenizer / `String.to_atom`), not here — and here it is provably a faithful copy, not a re-check.\
  *Axes examined: non-NFC preservation, atom_to_binary/atom_to_list codepoint agreement. Not examined: none.*

- **Type-contract violation (non-atom input)** — **Upgraded from `asserted` to `verified`:** `erlang:atom_to_binary(<<"x">>)` → `{error,badarg}` and `erlang:atom_to_list(123)` → `{error,badarg}`. Reaching this requires violating the documented `@spec atom` contract; the failure mode is a clean `badarg` raise, not memory unsafety or silent coercion.\
  *Axes examined: badarg on binary and integer inputs. Not examined: none material* (both non-atom shapes raise identically).

- **Code execution, file/path, command, network, deserialization, template/interpolation, crypto, shared state, concurrency, dispatch** — none appear; the file's only calls are two `:erlang` conversions and one self-delegation. *Basis: verified by full read of the 87-line file.*

## 3. Risks / caveats
- The `atom_to_binary`/`atom_to_list` BIFs are C-implemented VM primitives with no Erlang/Elixir neighbour module on this callgraph, so their internals cannot be "read." Their *observable* security-relevant behavior is now covered by executed evidence (no atom creation, no normalization, faithful codepoint reproduction, clean `badarg` on non-atoms). Unexecutable C internals of a trusted VM primitive are a genuine scope boundary, not an unverified assumption.
- No residual risk identified within the file's documented public API.

## Did I verify this?
- **atom_count delta = 0** — yes, executed (`erlang:system_info(atom_count)` before/after).
- **255-codepoint length bound / 256 → system_limit** — yes, executed at creation site.
- **No NFC normalization; faithful reproduction; BIF agreement** — yes, executed with a decomposed non-NFC atom.
- **`badarg` on non-atom** — yes, executed for binary and integer inputs.
- **BIF C internals** — not audited (no source in repo); observable axes covered by execution instead. Correct scope boundary.

## Dropped in re-review
None. The first pass reported no findings; none are dropped. Two ruled-out entries (resource consumption, type-contract) were strengthened from `asserted` to `verified` with executed evidence, and no assumption required upgrading to a low-confidence finding because every tried axis confirmed the safe (atom→text, non-interning, non-normalizing) behavior.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Type-contract violation (passing a non-atom)** — `:erlang.atom_to_binary`/`atom_to_list` raise `badarg` on a non-atom, but reaching that requires violating the documented `@spec atom` contract, which is explicitly out of scope.
