# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/dep/elixir_scm.ex
<!-- sha256: 7868d1dcb8c1151f171c1cec0b2ce05c3e938018b3ba94678c4c84c036dd39f9 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:13:12Z_

## Summary

`elixir_scm.ex` is a small internal (`@moduledoc false`) manifest reader/writer that serializes Erlang/OTP/Elixir/SCM/lock/deps state via `term_to_binary` and reads it back with a defensive `rescue` fallback; I found no genuine, reachable behavioural bugs.

## Bugfixes

No findings.

## Risks / caveats

- The review rests on the fact that this module is internal build-tooling depended on by `loadpaths.ex`, `deps.compile.ex`, `compilers/elixir.ex`, and `dep/loader.ex`. I read all four call sites and confirmed the return-shape and default-argument contracts line up.
- I could not construct any input reaching a crash or wrong value through these call sites; `read/1` deliberately catches every decode/match failure and returns a synthetic "old version" tuple that forces recompilation — verified below.

### Ruled out

- **Multi-default argument mis-binding in `update/4`** (`manifest_path \\ …, scm, lock, deps \\ []`). Elixir fills the *leftmost* optional first, so an arity-3 call `update(a, b, c)` would bind `manifest_path=a` (not the intended default). *Verified* with `elixir t.exs`: `T.update(:a,:b,:c)` → `{:a, :b, :c, []}`. However, both real callers use arity 2 (`compilers/elixir.ex:1010`) and arity 4 (`deps.compile.ex:205`) — grepped the whole `lib/mix/lib` tree; no arity-3 call exists — so the ambiguous binding is never triggered. Not reachable.
- **`read/1` crashing on corrupt/empty/old-version manifests.** *Verified* with a standalone reproduction of the `read` body: empty binary, a garbage term (wrong shape), and an old `@manifest_vsn` all return the fallback `{:ok, {"1.0.0", ~c"17"}, nil, nil, []}` rather than raising; a well-formed manifest round-trips correctly. The `rescue _` catches both `binary_to_term`'s `ArgumentError` and the tuple `MatchError`, which is the intended "force recompile" behaviour (consistent with the `old_vsn != vsn` guards in `loadpaths.ex:62` and `loader.ex:432`).
- **Unsafe `binary_to_term` (no `:safe`) enabling atom-table exhaustion / decode attack.** The manifest lives under the local build directory and is written only by `update/1` on the same machine; Hex/Git dependencies ship source, not compiled build manifests, so no untrusted binary reaches this decode through the documented flow. Checked that the only writer is this module's own `term_to_binary` call. Not reachable from untrusted input.
- **`rescue` swallowing `throw`/`exit`.** `binary_to_term` and the tuple match raise exceptions (`ArgumentError`/`MatchError`), not throws/exits, so `rescue` is the correct construct here. No asymmetry.

