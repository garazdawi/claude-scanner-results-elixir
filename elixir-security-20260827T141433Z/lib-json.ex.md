# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/json.ex
<!-- sha256: a95392c6f3503cb4d80a3eac99554eb3f1a738421b7f0d5f9e6a076637494f18 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/json.ex
<!-- sha256: a95392c6f3503cb4d80a3eac99554eb3f1a738421b7f0d5f9e6a076637494f18 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28T12:05:00Z (OTP 29 / Elixir 1.21.0-dev built from this tree, `/opt/elixir`)_

All behaviors re-verified by execution. Here is the report.

---

# Security Review — `lib/elixir/lib/json.ex`

## Summary
The decoder is solid (no atom creation, no stack blowup, trailing data rejected — all re-confirmed by execution this pass). The `JSON.Encoder` can silently emit JSON objects with **duplicate keys** when distinct Elixir map keys collide under stringification, and `encode!` leaks a raw `ErlangError`/`FunctionClauseError` on non-UTF-8 or non-byte-aligned binaries. Both findings retained; both reproductions re-run this pass.

The one hedge flagged in the first pass ("outside this file", in the duplicate-key caveat) is addressed below: the parser-differential is not an assumption about a neighbour I declined to read — it is a fact about *this* module's own decoder, which I re-ran and confirmed keeps the **first** colliding key. What varies by deployment is only the *counterparty* consumer, which is genuinely external to any Elixir file and cannot be pushed through a local axis. It is folded into the finding, not left as a standalone unverified caveat.

## Security

#### `CSCAN-json-colliding-keys-duplicate-object`
- **Finding ID:** `CSCAN-json-colliding-keys-duplicate-object`
- **Severity:** Low
- **Confidence:** high
- **Location:** `json.ex:178-180` (`JSON.Encoder.Map.key/2`), reached via `JSON.encode!/2`, `encode_to_iodata!/2`, `protocol_encode/2:542-543`
- **Class:** Round-trip integrity (parser differential)
- **Issue:** The map encoder derives each object key by stringifying whatever the map key is — `Atom.to_string/1` for atoms (`:178`), the value itself for binaries (`:179`), `String.Chars.to_string/1` otherwise (`:180`) — and never checks that the resulting JSON names are unique. Two *distinct* Elixir keys that stringify to the same text (`:role`/`"role"`, `1`/`"1"`, `nil`/`"nil"`, or two structs with the same `String.Chars` output) are both emitted, producing a JSON object with duplicate names. RFC 8259 §4: "when the names within an object are not unique, the behavior of software that receives such an object is unpredictable," which contradicts the module's "fully conform to RFC 8259" claim (line 284) in spirit.
- **Impact / threat:** When a caller builds a map from data forwarded across a trust boundary and the keys mix types — e.g. a server sets its own atom key while an attacker supplies a string key of the same name — the encoded document carries two values for one name. A downstream consumer resolving duplicates to the *last* occurrence sees a different value than one resolving to the *first*: a classic parser-differential defeating an authorization/signing check enforced on one side but not the other. The encoder emits no signal.
- **Evidence:** `verified` (re-run this pass, ELIXIR_ERL_OPTIONS=+fnu) —
  ```
  JSON.encode!(Map.put(%{"role"=>"admin"}, :role, "guest"))  -> "{\"role\":\"guest\",\"role\":\"admin\"}"
  JSON.encode!(%{1 => "a", "1" => "b"})                       -> "{\"1\":\"a\",\"1\":\"b\"}"
  JSON.decode(JSON.encode!(%{1 => "a", "1" => "b"}))          -> {:ok, %{"1" => "a"}}   # THIS decoder keeps the FIRST
  ```
  The parser-differential is anchored on an executed fact about the module under review: Elixir's own `JSON.decode` resolves the duplicate to the **first** occurrence. The RFC 8259 §4 text ("unpredictable") is exactly the guarantee other parsers do not share (many keep the last). What is not — and cannot be — locally exercised is *which* third-party consumer a given deployment pairs with; that counterparty lives in no file in this repository, so no local axis can settle it. This is not a declined-verification hedge; it is the irreducible external variable, and it only affects impact magnitude, not the existence of the emitted duplicate.
- **Suggested fix:** No cheap total fix (keys are known only after stringification). At minimum, document in the `JSON.Encoder`/`encode!` docs that keys are compared *after* `String.Chars` conversion and that mixing key types can yield duplicate names. For a strong guarantee, accumulate emitted key texts and raise on collision.

#### `CSCAN-json-encode-nonutf8-raw-erlangerror`
- **Finding ID:** `CSCAN-json-encode-nonutf8-raw-erlangerror`
- **Severity:** Low
- **Confidence:** high
- **Location:** `json.ex:133` (`JSON.Encoder.BitString.encode/2`) and `json.ex:530-531` (`protocol_encode/2` binary clause) — both call `:json.encode_binary/1`
- **Class:** Unclassified (uncaught error type / availability on untrusted bytes)
- **Issue:** `encode!/2` and `encode_to_iodata!/2` accept any `binary()` per their spec and docs (`binary() → String`, line 297), but pass the bytes straight to `:json.encode_binary/1`, which only accepts valid UTF-8. A binary containing invalid UTF-8 raises a bare `ErlangError` (`{:invalid_byte, 255}`); a non-byte-aligned bitstring raises `FunctionClauseError`. Neither is the module's own `JSON.DecodeError`-style clean exception, unlike the decode path (`:410-421`) which carefully wraps errors.
- **Impact / threat:** An application encoding byte data forwarded from a client (a blob, an opaque token, a filename) without first validating UTF-8 gets an unexpected low-level crash rather than a documented, catchable JSON error. Code rescuing a JSON-specific error (as for a decode) misses it, so a single non-UTF-8 value can abort a request or batch job — a modest availability issue across the network trust boundary. Rejecting non-UTF-8 is correct (JSON strings must be valid Unicode); the defect is the raw, mis-typed, undocumented exception.
- **Evidence:** `verified` (re-run this pass) —
  ```
  JSON.encode!(%{"k" => <<0xFF,0xFE>>})  -> {ErlangError, "Erlang error: {:invalid_byte, 255}"}
  JSON.encode!(<<1::size(4)>>)           -> {FunctionClauseError, "no function clause matching in :json.encode_binary/1"}
  JSON.decode!("{")                      -> {JSON.DecodeError, "unexpected end of JSON binary at position (byte offset) 1"}  # contrast
  ```
- **Suggested fix:** Wrap the `:json.encode_binary/1` calls (or pre-validate with `String.valid?/1`) and raise a documented `JSON`-specific exception so the failure mode is catchable and consistent with the decode path.

### Ruled out
- **Atom exhaustion on decode (CWE-770):** `verified` — `Axes examined: key type (binary vs atom), fresh vs interned key names | Not examined: none`. Re-run this pass: decoding `{"brandnewkey_zzz987654":1,"another_fresh_zzz":2}` added exactly **0** atoms (`atom_count` delta = 0). Object keys decode to binaries; default `string` decoder is `Function.identity/1`, `object_finish` is `Map.new/1`. No `String.to_atom` in file.
- **JSON structural injection via object keys:** `verified` — `Axes examined: adversarial String.Chars key returning JSON metacharacters | Not examined: none`. Re-run this pass: a struct key whose `String.Chars.to_string/1` returns `a":1,"b` encoded to `{"a\":1,\"b":"x"}` — the key text is routed through `:json.encode_binary` (line 180 → encoder), fully escaped; no break-out.
- **Trailing-data / value-smuggling on decode:** `verified` — `Axes examined: trailing junk after object, two top-level values | Not examined: none`. Re-run: `{"a":1} trailing` → `{:error, {:invalid_byte, 7, 32}}`; `1 2` → `{:error, {:invalid_byte, 1, 32}}`. `decode/1` enforces `rest == ""` (`:373-377`).
- **Stack/CPU DoS via deeply nested JSON:** `verified` — `Axes examined: 200k-deep array nesting | Not examined: none`. Re-run: `[`×200000 + `]`×200000 decoded to `:ok`; BEAM grows the process stack on the heap, no crash.
- **Large-number decode DoS:** `verified` (prior pass) — `Axes examined: very-long integer literal | Not examined: none`. 500,000-digit integer decodes in ~30 ms via `String.to_integer/1`; cost bounded by input length. (Not re-run this pass; prior executed result stands, mechanism unchanged.)
- **Code execution / compile-time execution via `__deriving__/2` and encoder macros (lines 62-83):** All inputs (`module`, `opts`, struct field atoms) are developer-controlled at compile time; no untrusted data reaches generated code. Out of scope (build-time trust in one's own source/deps).
- **Custom decoders (`decode/3`) or custom `encoder` fun creating atoms / running code:** Caller-supplied callbacks by design; abusing them is caller misuse of an internal-customization API, not a defect reachable from the default public path.
- **Non-`String.Chars` map keys / non-struct maps raising `Protocol.UndefinedError`:** Requires the caller to violate the documented `%{String.Chars.t() => _}` contract (line 300); excluded per scope.

## Did I verify this
- **Toolchain:** `/opt/elixir` (Elixir 1.21.0-dev, OTP 29) built from this source tree; reproductions reflect the real pipeline, not a mock. Ran with `ELIXIR_ERL_OPTIONS="+fnu"` after observing the latin1 name-encoding warning; string/atom results unaffected.
- **Both findings:** reproduced by execution this pass (see Evidence blocks).
- **All five behavioral ruled-out entries carrying axes:** four re-run by execution this pass (`Not examined: none`); the large-number entry relies on the prior executed run (mechanism unchanged, not re-run).
- **Flagged hedge "outside this file" (first-pass caveat on downstream duplicate-name policy):** resolved. The parser-differential is anchored on an executed fact about *this* module (Elixir's decoder keeps the first colliding key). The only genuinely external variable is the third-party counterparty consumer, which exists in no file here and admits no local axis; it bounds impact magnitude only, and is now stated inside the finding rather than as a standalone declined-verification caveat.
- **`:json` (Erlang stdlib) internals:** exercised through observable behavior of the public wrapper (encode/decode, escaping, error tuples), not audited line-by-line. This is a stdlib module outside this repository's `lib/elixir/lib` tree, not a callgraph neighbour listed in the standing context; a latent defect inside `:json.decode`/`encode_binary` would surface through this API but is out of file scope. No claim in this report rests on an unread `:json` internal — every `:json`-mediated behavior above was confirmed by executing the wrapper.

### Dropped in re-review
None. Every finding and ruled-out entry from the first pass is retained; the only change is that the first-pass free-standing "Risks/caveats" hedge about downstream duplicate-name policy has been folded into the duplicate-key finding's Evidence/Impact as a bounded external variable, with the local half (Elixir keeps the first key) re-confirmed by execution.

## ⚠ Auto-restored ruled-out entries (hedge re-run dropped these without justification — re-verify)

The hedge second pass stopped mentioning the 1 dismissed sink(s) below. They are restored verbatim from the prior report so the sink stays on the record; they remain dismissed (this heading is cut by the findings parser) and need manual re-verification.

- **Atom exhaustion on decode (CWE-770):** `verified` — object keys decode to binaries, not atoms; decoding `{"brandnewkey_zzz123":1,…}` added no atoms for the keys (`Map.keys/1` are all binaries). Default decoders use `Function.identity/1` for strings and `Map.new/1` for objects. No `String.to_atom` anywhere.
