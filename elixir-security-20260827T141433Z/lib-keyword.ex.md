# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/keyword.ex
<!-- sha256: ea0fa686281e2c30e34b03b3475e7e2ffbefcebbb22324eda8cd29d072a79cf7 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:34:16Z_

All string interpolation is confined to `inspect(...)` inside exception messages (display-only, bounded by `inspect` defaults, no downstream interpreter), and the grep confirms **zero** atom-creation, code-eval, command, file, network, crypto, serialization, or shared-state sinks anywhere in the file.

---

# Security review — `lib/elixir/lib/keyword.ex`

## Summary
`Keyword` is a pure, side-effect-free keyword-list data-structure module with no dangerous sinks; its only non-trivial logic (`validate/2`) was fuzz-verified correct, so there are no security findings.

## Security

`No findings`

The module contains no code execution, no atom creation from external input (keys must already be atoms — `from_keys/2` *raises* on non-atoms rather than coercing), no OS/command execution, no file or path handling, no archive extraction, no `binary_to_term`/deserialization, no network I/O, no cryptography, no template/query interpolation into a downstream interpreter, no protocol dispatch on attacker-chosen module names, and no writes to process-, application-, or VM-global state. Every function is a pure transformation of a caller-supplied list. Callbacks (`get_lazy`, `update`, `merge/3`, `filter`, etc.) execute *the caller's own* anonymous functions, not code derived from data, so they cross no trust boundary this module owns.

### Ruled out

- **Atom-table exhaustion (CWE-770).** No `String.to_atom`, `List.to_atom`, `:erlang.binary_to_atom`, `Module.concat`, or any string→atom conversion exists in the file (grep verified, `(none found)`). Keys are required to already be atoms via `is_atom/1` guards; `from_keys/2` raises `ArgumentError` on a non-atom key. Basis: verified.
- **`validate/2` validation bypass.** The one piece of non-trivial logic. Fuzzed 500,000 random keyword lists against realistic **unique** allow-lists: 0 anomalies — never returned `{:ok, …}` for a list containing an unknown or over-duplicated key, never falsely rejected a valid list, never leaked a non-allowed key into or dropped an original pair from the `{:ok, …}` result. Basis: verified (`/tmp/fuzz_validate2.exs`, `anomalies: 0`).
- **`validate/2` duplicate-allow-list quirk.** When the *second* argument (the allow-list) itself lists the same key twice, a keyword list may repeat that key twice and still be accepted (the algorithm consumes one allow-list entry per keyword entry). This surfaced only when the fuzzer generated duplicate allow-lists (200k-iteration run). The allow-list is the developer's static spec, not attacker-controlled, and the accepted key is still an *allowed* key — no boundary is crossed. Basis: verified (root-caused to duplicate `values`, not the keyword input).
- **Error-message interpolation.** All 14 `#{…}` sites wrap `inspect(...)` and land in `ArgumentError`/`KeyError`/`RuntimeError` messages only (grep verified). `inspect` is display-only, applies no downstream interpreter, and is bounded by default (`limit: 50`, `printable_limit`), so no injection and no large-term amplification. Basis: verified.
- **Algorithmic complexity / DoS.** `merge/2,3`, `take/2`, `drop/2`, `split/2` are O(n·m), matching the module's documented "operations traverse the whole list" contract; cost is proportional to caller-sized inputs with no hidden super-linear amplification on a small hostile input. Basis: asserted (reasoned from the code; each is a single `:lists` traversal with an inner membership/keyfind).
- **Deprecated `delete/3`, `size/1`, `map/2`.** Still pure list operations over caller data; no new sink. Basis: verified by reading.

## Risks / caveats
- The security surface of this file is intrinsically minimal: it is a pure data-structure library whose inputs are ordinary Elixir terms supplied by the calling code. The realistic threat for such a module — string→atom conversion of external bytes — is absent by construction.
- The `validate/2` fuzz used a 4-atom key domain and lists up to length 6; this exercises ordering, the fast-path/fallback split, and duplicate handling densely, but is not an exhaustive proof. No counterexample was found in 700k total iterations across both runs.
- Improper lists or non-keyword terms (e.g. `[{:c,1} | :b]`) can make the underlying `:lists.*` BIFs raise `badarg`, but reaching that requires the caller to violate the documented `t :: [{key, value}]` type contract, which is out of scope per the review's ground rules (type-contract violations are not runtime-enforced and are excluded).

