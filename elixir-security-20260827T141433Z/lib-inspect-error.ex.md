# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/inspect/error.ex
<!-- sha256: ae1babeaa7736c0d86e689fc5592430b011d6b698579672d9e980fd34173956c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:07:24Z_

I've completed my analysis. Let me write the report.

---

## Summary

`Inspect.Error` is a small, purely-formatting stdlib module (build an exception struct, render a human-readable message/inspection) with no code-execution, atom-creation, file, network, or deserialization sinks, and its one notable behavioral quirk (its `Inspect` impl ignores `opts`) is by-design and produces bounded output for any data-only attacker.

## Security

No findings.

The file's entire attack surface is text formatting of an already-materialized error. I traced every value that enters through the documented public API (`exception/1`, `message/1`, and the `Inspect` protocol impl) and found no sink where untrusted **data** (as opposed to attacker-supplied *code*) crosses a trust boundary into a dangerous operation.

### Ruled out

- **`Inspect` impl ignores `_opts` → `:printable_limit`/`:limit` not enforced (output-size amplification).** The `defimpl Inspect` at `error.ex:64` renders the full message regardless of the caller's limits. **Basis: verified** — `inspect(boom, printable_limit: 10, limit: 5)` produced **200,648 bytes** when a custom Inspect impl raised `ArgumentError` with a 200 KB string. However, the trust boundary here is *data*, not *code*: reproducing the amplification with **data-only** triggers (invalid stdlib structs built via `struct/2`, and a `KeyError` carrying a 100k-entry map) yielded **bounded** output (702 B for a bad `Date`, 896 B for a bad `Time`, ~8.9 KB for the big `KeyError`), because `inspected_struct` is built by `Inspect.Map.inspect_as_map` under the *same* `limit`/`printable_limit` and stdlib exception messages self-truncate via default inspect limits. The unbounded case requires the victim to have loaded a struct whose Inspect implementation *raises an attacker-sized message* — i.e. attacker-controlled code — which is excluded by the threat model (analogous to "victim runs `eval` on attacker input"). Ignoring opts is also the module's stated purpose (surface the full failure for debugging).
- **`pad/2` quadratic/DoS (`error.ex:47`).** **Basis: verified** — timed on 50k→400k-line inputs (450 KB→3.6 MB output): runtime scales ~linearly (4.8 ms→71 ms) with output size, no quadratic blowup. Its input is a bounded diagnostic string, and it is `@doc false` (not public API).
- **Diagnostic-string "delimiter"/log spoofing.** `message/1` (`error.ex:33`) interpolates the raw, unescaped `exception_message` between `"""` markers, so text in it could visually fake message boundaries. Not a security sink: the output is a developer-facing/log string, not fed to any downstream interpreter (no SQL/shell/EEx/HTML), `inspected_struct` is already inspect-escaped, and influencing `exception_message` content requires attacker code.
- **Recursion via `message/1` → `inspect(exception_module)` (`error.ex:34`).** Deep recursion would require `exception_module` to be a nested `%Inspect.Error{}` rather than the atom the normal `exception/1` path assigns (`exception.__struct__`), i.e. a direct, contract-violating struct construction; **verified** a hand-built nested value still inspected to 225 bytes, and any chain costs O(depth) to construct in the first place — no amplification.
- **Atom exhaustion.** None — `exception.__struct__` (`error.ex:14`) reads an existing module atom; there is no `String.to_atom`/`Module.concat`/`binary_to_atom` in the file.
- **Secret-redaction bypass on the inspect-failure path.** The concern (a `@derive {Inspect, except: [...]}` struct falling back to raw map rendering that shows the redacted field) lives in `Inspect.Algebra.to_doc_with_opts` (`algebra.ex:411/417`, which calls `Inspect.Map.inspect_as_map`), **not** in this file — `error.ex` only formats the already-built `inspected_struct` string. Additionally **verified** that a derived redacting impl is total (does not raise), so it does not reach the failure path under normal use.

## Risks / caveats

- **Scope boundary.** `inspected_struct` is constructed by the caller in `Inspect.Algebra` (`algebra.ex:398–450`), which is outside this review. If a future change let that string carry un-truncated attacker data or bypass field redaction, the amplification/leak surface would grow — but the defect and fix would belong to `algebra.ex`, not `error.ex`. I confirmed today's behavior is bounded and non-leaking for the data-only case.
- **The threat model excludes attacker-controlled code.** The only way I could make output unbounded was via a custom `Inspect` implementation that itself raises an attacker-sized message; that is a code-trust issue, not reachable from untrusted data through this file's public API. If a specific deployment treats third-party *Inspect implementations* as untrusted, revisit the opts-ignoring behavior — but that is a design decision spanning the whole inspect subsystem, not a bug localized here.
- **Contract-violating direct construction** (e.g. `%Inspect.Error{stacktrace: not_a_list}`, or `exception_module` set to a non-atom) can crash or recurse, but this requires bypassing the documented `exception/1` constructor and is excluded by the ground rules (caller violating the type contract).

