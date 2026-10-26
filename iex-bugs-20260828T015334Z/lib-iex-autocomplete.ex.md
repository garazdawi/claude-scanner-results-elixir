# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/autocomplete.ex
<!-- sha256: 4725cb1ead57e722ffb4650137c9d5b91d8da0a24e653fd2d3f23cb375ca168f -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:06:30Z_

I've read the full file, run its expand logic across a broad set of completion inputs without crash, and traced the parsing/formatting hot paths. Here is my report.

## Summary
After a full read, targeted reasoning, and fuzzing `expand/2` across dozens of completion fragments, I found no real, triggerable correctness bug reachable through this module's documented public API.

## Bugfixes

No findings

### Ruled out

- **`format_expansion/2` else-branch `binary_part(first.name, prefix, length - prefix)` with a negative length (server.ex-style off-by-one)** — Since every caller filters entries so all `name`s start with `hint`, the longest common `prefix` is always ≥ `length = byte_size(hint)`, and reaching the else branch means `prefix > length`. `binary_part/3` with a negative length is defined (returns bytes `[length, prefix)`), which is exactly the intended completion extension. Verified: `expand(~c"En")` → `{:yes, ~c"um", []}` (hint len 2, common prefix "Enum" len 4, `binary_part("Enum",4,-2) = "um"`), matching the test suite.

- **`t`/`b` typespec helper suppressing module (alias) completion** — I suspected the `expansion when helper == ?b/?t` guards (lines 86–90) would shadow alias completion. They do not: the `{:alias, alias}` clause (line 77) precedes them, so `t String`/`b String` still complete module names; the helper guards only intercept `{:dot, …}` (line 92). Confirmed by tests at `autocomplete_test.exs:61-89` (`t String` → `{:yes, ~c"", [~c"String", ~c"StringIO"]}`).

- **`path_fragment/2` not completing bare relative paths (e.g. `"foo/bar`)** — The state machine only recognizes `"/…` (absolute), `"./…`, and `"X:/…` (drive); a bare opening `"` hits `path_fragment([?"|_],_) -> []`. This is intended behavior (avoid treating every string literal as a path), matching `autocomplete_test.exs:578-603` which only asserts completion for `"./`, `"/`, and the absolute tmp dir. Escaped-quote handling (`[?", ?\\ | t]`) correctly precedes the terminating `[?" | _]` clause.

- **`recur_expand_dot_path({:alias,…})` returning `{:ok, atom}` while `{:var,…}` returns `{:ok, [list]}`** — This asymmetry is deliberate: `expand_dot_path` matches `{:ok, [_|_]}` for the binding path and lets the atom fall through as `other`, delivering `{:ok, atom}` to `expand_dot`'s `is_atom` clause. The `{:dot, parent, call}` recursion returning `:error` for an alias parent is correct — chaining field access off `Alias.fun` (a call, not a value) is not statically completable.

- **`match_modules_from_applications/1` `{:ok, modules} = :application.get_key(app, :modules)` raising `MatchError`** — `loaded_applications/0` yields only apps in `:ac_tab`'s `{:loaded, app}` state; OTP `application:get_key/2` returns `{:ok, Value}` (with `:modules` defaulting to `[]`) for any loaded application and `:undefined` only for unloaded ones, so the match cannot fail for these inputs.

- **`expand_sigil/0` passing `hint = "~"` to `format_expansion` with entries whose names (e.g. `"w"`, `"s"`) don't start with `"~"`** — Potentially reaches `format_hint` with `hint_size > byte_size(name)` (a negative `binary_part` length). Not triggerable: Kernel always provides many sigils, so the multi-entry branch is taken, the common prefix collapses to `0` (∈ `[0, length]`), and the group path returns all entries without ever calling `format_hint`. The single-entry/empty-name path would require a lone imported function literally named `sigil_` and no other sigils — unreachable in a real session.

- **`get_and_match_module_defs/3` `cond` treating an empty docs list (`[]`, truthy in Elixir) as "has docs"** — Both the docs branch and the `exports/match_exports` branch produce equivalent results for a module with an empty doc set, so no observable defect.

- **`GenServer`/message-handling checks from the folder threat model** — Not applicable: this module has no process, `receive`, or `handle_info`; it is pure completion logic invoked synchronously.

## Risks / caveats
- I could not execute the module's ExUnit suite end-to-end (the sandbox build/tmp directories are read-only: `test_helper.exs:8` fails on `/cscan/0/tmp/beams`). I compiled the `iex` app successfully and independently exercised `expand/2` via a standalone script against a live evaluator, so behavioral claims above are verified against real execution, but full-suite green is asserted from upstream provenance, not observed here.
- Review was scoped to `autocomplete.ex`. Cross-module contracts it depends on (`IEx.Broker.evaluator/1`, `IEx.Evaluator.*_from_binding`, `Code.Fragment.cursor_context/container_cursor_to_quoted` return shapes) were assumed to honor their documented forms; a regression in those could surface here but belongs to those files' review.
- The remsh path (`:erpc.call` with a locally-resolved `:shell.whereis()` pid used on the remote node) is the documented distributed-shell feature and was treated as in-contract rather than audited as a trust boundary.

