# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/access.ex
<!-- sha256: 718a27fb1b28b94563b08770270076b20adfa563a09ff8c4784d9a79c3551096 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T08:18:10Z_

I have completed a thorough review, running the closest-to-findings edge cases to verify behavior. Here is the report.

---

# Security Review — `lib/elixir/lib/access.ex`

## Summary
`Access` is a pure key-based data-structure access module with no dangerous sinks (no code evaluation, atom creation, file/OS/network access, or deserialization), and its only externally-influenced behaviors — struct-tag dispatch and list/range traversal — are by-design and bounded, so no meaningful security issues were found.

## Security

`No findings`

### Ruled out

- **Behaviour dispatch on a caller-controlled struct tag (`fetch/2`, `get/3`, `get_and_update/3`, `pop/2`).** The clauses `def fetch(%module{} = container, key)` etc. (access.ex:251, 326, 402, 468) invoke `module.fetch(container, key)` where `module` is the map's `__struct__` value. This is a genuine dispatch on an externally-influenced type, but it is the documented, intended semantics of the Access protocol (`data[key]`/`get_in` on structs must call the struct's implementation). Exploiting it as a gadget requires the attacker to already control an in-process term with a chosen — and already-existing — module atom in `__struct__`; that primitive (e.g. unsafe `binary_to_term`) is itself the vulnerability and is far more powerful. The callable surface is narrow (functions literally named `fetch`/`get_and_update`/`pop` on loaded modules) and this file adds no unsafe atom creation (`module` is destructured from an existing tag, never built from bytes). *Basis: verified* — `Access.fetch(%{__struct__: Enum, a: 1}, :a)` dispatched into `Enum.fetch/2` (raising `FunctionClauseError`), confirming dispatch follows the tag with no new capability introduced here.
- **`is_probably_keyword/1` heuristic guard (access.ex:229) as a validity gate.** Used in `key/2` and `key!/1`, it only inspects the first list element (`is_atom(elem(hd(list), 0))`). It is not a safety predicate — it merely selects the keyword branch vs. the error branch. Malformed lists degrade to a clean `RuntimeError`, not a crash or a bypass. *Basis: verified* — `get_in([1,2,3], [Access.key(:a)])` and `get_in([:a,:b], [Access.key(:a)])` both raised `RuntimeError "Access.key/2 expected a map/struct/keyword list…"` (the `elem/2` failure inside the guard safely makes the guard fall through).
- **Resource consumption in `slice/1`, `at/1`, `at!/1`, `all/0`, `filter/1`, `find/1`, `values/0`.** All traversals are linear in the input data structure; large or extreme range/index bounds do not translate into work proportional to the bound. `slice/1` rejects non-positive steps at construction (access.ex:1023), and recursion terminates on list exhaustion, not on the numeric bound. *Basis: verified* — `Access.slice(0..1_000_000_000)` and `Access.at(1_000_000_000)` / `Access.at(-1_000_000_000)` over a 3-element list each returned in ≤20µs, bounded by list length.
- **Error-message interpolation (`raise "… got: #{inspect(data)}"`, multiple sites).** Only `inspect/1` reaches the string, which does not evaluate its argument and truncates by default (`limit: 50`); the term is caller-supplied and this is the error path. No downstream interpreter (shell/query/markup/regex) is involved. *Basis: file read in full.*
- **Caller-supplied callbacks (`next`, `func`, `predicate`) invoked in `all/filter/find/at/values`.** These are the accessor/`get_in` chain the caller itself constructs, not external input; invoking them is the API contract, not injection. *Basis: file read in full.*
- **Absent sink classes.** A complete read of the file confirms no `Code.eval*`/`Module.create`, no `String.to_atom`/`binary_to_atom`/`Module.concat`, no `binary_to_term`, no `System.cmd`/`os.cmd`/`System.shell`, no `File`/`Path`/`:erl_tar`/`:zip`, no `Regex.compile`, no `:crypto`, and no writes to ETS / `:persistent_term` / `Application` env / process dictionary. *Basis: file read in full.*

## Risks / caveats
- The behaviour-dispatch "gadget" ruling assumes the standard Elixir threat model in which an attacker controlling in-process terms with arbitrary struct tags already has an equal-or-greater primitive; if a specific downstream caller feeds attacker-chosen `__struct__` maps into `Access.*` without that being the case, the dispatch is a pre-existing property of Access to be reviewed at that call site, not a defect in this file.
- Review was scoped to this single file; the behavior of the `Map`/`Keyword`/`Enum`/`Range` functions it delegates to was assumed correct and not independently audited.
- Verifications ran against Elixir 1.21.0-dev built from this tree (Erlang/OTP 29); behavior on other versions was not checked, though this module's logic is version-stable.

