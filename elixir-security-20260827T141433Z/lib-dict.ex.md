# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/dict.ex
<!-- sha256: 8ccdc914c2f91ab893e93dfd790087000d3135587cad4bbb00d82dd43bf73591 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T10:37:13Z_

I have fully characterized the module's behavior. Writing up the report.

---

## Summary

`Dict` is a deprecated, purely polymorphic dispatch shim (tokenizer/decoder/atom/eval/file/network sinks are all absent); its only security-relevant pattern — dispatching to a module named by a map's `__struct__` field — is by-design polymorphism, constrained to a fixed set of benign dict-shaped function names, and I found no meaningful vulnerability reachable through its public API.

## Security

`No findings`

### Ruled out

- **Module dispatch on caller-controlled `__struct__` (`target/1`, dict.ex:243‑252).** Every public function routes through `target(dict)`, which for a struct binds `module` from `dict.__struct__` and calls `module.keys/1`, `module.get/3`, `module.merge/2`, etc. directly (not via a protocol table). A crafted plain map such as `%{__struct__: Evil}` does dispatch to `Evil.get/3` — **verified**: `elixir /tmp/dict_test.exs` printed `>>> Evil.get/3 was invoked via Dict dispatch <<<` and returned `:pwned`. I ruled this out as a finding *for this file* because: (a) it is the documented, intended purpose of `Dict` (a generic API over any dict-like module — `HashDict`, `Map`, `Keyword`); (b) the reachable function names are a fixed, dict-shaped set (`get/put/fetch/keys/values/delete/merge/pop/…`), so an attacker cannot steer dispatch to `Code.eval_string`, `File.rm_rf`, or any non-dict sink — the gadget surface is benign by construction; (c) it is only "reachable with hostile input" if a caller *first* deserializes untrusted structs (e.g. `:erlang.binary_to_term` without `:safe`) and forwards them here — that is the caller's vulnerability, and the same crafted struct would misbehave in `Enum`, `Access`, etc.; and (d) the module has been deprecated since Elixir 1.2. Basis: **verified**.
- **Crash / DoS via a bogus `__struct__` atom.** A `__struct__` naming a loaded module without the expected function, or an unloaded module, raises a clean `UndefinedFunctionError` rather than corrupting state or hanging — **verified** (`dict_test2.exs`: cases A and B both → `UndefinedFunctionError`). A non-atom `__struct__` does not even match `%module{}` (it falls to the `%{}` → `Map` branch), so no atom-type confusion — **verified** (case: `non-atom __struct__ -> :plain_map`). No unbounded resource use.
- **Atom creation.** None. `target/1` reads an already-existing atom out of a struct; there is no `String.to_atom`, `binary_to_atom`, or `Module.concat` in the file.
- **Charlist vs. keyword-list branch confusion (`is_list(dict) -> Keyword`, dict.ex:248).** Passing a non-keyword list (e.g. a charlist) routes to `Keyword`, which tolerates it and returns `nil` rather than crashing — **verified** (`Dict.get(~c"hi", :k)` → `nil`). This requires the caller to violate the documented `t :: list | map` (keyword-list) contract and yields no memory-unsafety or injection; excluded per the contract-violation scope rule.
- **`inspect(dict)` in `unsupported_dict/1` (dict.ex:445).** Interpolates the offending term into an `ArgumentError` message via the safe `Inspect` protocol; the term is the caller's own argument, not a cross-boundary secret. No injection sink (not a shell/query/format-string interpreter).
- **`__using__/1` compile-time macro (dict.ex:25‑241).** Runs at build time but takes no external input — argument is ignored (`_`), and the only interpolated value (`unquote(message)`) is a hardcoded literal. The generated functions invoke caller-supplied `fun.()` closures, which is ordinary higher-order behavior, not code injection.
- **Code execution / command exec / path handling / deserialization / crypto / network / template / shared-state sinks.** None present — the file contains no `Code.*`, `System.*`, `:os.cmd`, `File.*`, `Path.*`, `:erlang.binary_to_term`, `EEx`, `Regex` interpolation, `Base`/`URI` decode, `:crypto`, `Logger`, `:ets`/`:persistent_term`/`Application.put_env`/`Process.put`, or any compile-time filesystem/network read. Basis: full read of all 448 lines plus keyword scan.

## Risks / caveats

- The dispatch gadget is real and verified; my "ruled out" verdict is a judgment that it is *by-design and benign-by-construction for this file*, not that dispatch cannot occur. If a downstream consumer feeds attacker-controlled deserialized structs into `Dict.*` **and** a loaded module happens to export a dict-named function with a dangerous side effect, that specific composition could be abused — but the defect there lives at the deserialization boundary and in that gadget module, not in `dict.ex`.
- I did not enumerate the full set of currently-loaded modules exporting `get/3`, `put/3`, `delete/2`, etc. with harmful side effects; the fixed dict-shaped naming makes a dangerous match unlikely, but this is an assertion about a naming convention, not an exhaustive proof over every BEAM library.
- All reproductions ran against Elixir 1.21.0-dev / OTP 29 built from the source under review; behavior on other versions was not tested. The environment's latin1 native-encoding warning did not affect the ASCII-only test inputs used.

