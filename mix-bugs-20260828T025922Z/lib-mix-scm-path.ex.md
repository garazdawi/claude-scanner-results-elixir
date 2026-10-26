# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/scm/path.ex
<!-- sha256: 3c70e7b6b21935a37ef5c5b91a635fd43b2879f64fdc00433e30e1df59f497fd -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:25:12Z_

All callbacks behave exactly as the `Mix.SCM` behaviour documents. The file is a straightforward, correct SCM implementation.

## Summary

`Mix.SCM.Path` is a small, `@moduledoc false` implementation of the `Mix.SCM` behaviour for path dependencies; every callback matches the behaviour's documented return shape and I found no genuine, reachable defect.

## Bugfixes

No findings

## Risks / caveats

- The file has no `@spec`s of its own; it relies on `@impl true` against `Mix.SCM`'s `@callback`s, and I confirmed each implemented arity matches a declared callback (`fetchable?/0`, `format/1`, `format_lock/1`, `accepts_options/2`, `checked_out?/1`, `lock_status/1`, `equal?/2`, `managers/1`, `checkout/1`, `update/1`). The folder threat model's listing of `lock_status/2`/`managed_deps/0` is stale relative to the actual `scm.ex` in this tree (`lock_status/1`, `managers/1`), which I read directly.
- I exercised the public callbacks against the source under review (see the verified run above); behaviour matched the docs in every case. I did not run the full Mix dependency-resolution pipeline that *calls* these callbacks, so higher-level interaction bugs (which would live in `Mix.Dep.*`, not this file) are out of scope and unverified here.

### Ruled out

- **`format/1` could return `nil` when `opts[:path]` is absent, violating the `@callback format(opts) :: String.t()` shape.** Considered as a contract mismatch. Ruled out: reaching it requires an `opts` with no `:path` key, but `accepts_options/2` (the only sanctioned producer of these opts) guarantees `:path` is present in both the explicit-path branch (matched on `opts[:path]`) and the in-umbrella branch (`Keyword.put_new(:path, "../#{app}")`). A well-behaved caller using the documented API never reaches `format/1` with a path-less opts. Verified `format(path: "abc")` returns `"abc"` in the run above.
- **`accepts_options/2` returning `nil` for the empty-options case could crash a caller expecting opts.** Ruled out: `nil` is the documented return for "SCM does not consume this dependency" (`@callback accepts_options(app, opts) :: opts | nil`), and callers try each SCM in order. Verified `accepts_options(:foo, [])` returns `nil`.
- **`checkout/1` always raising instead of returning a lock.** Ruled out: this is the intended contract for a non-fetchable path dependency (`fetchable? == false`) — a path dep that is not already on disk is a hard user error, and `Mix.raise` is the correct surfacing. The message includes the expected path via `Path.relative_to_cwd/1`, so it is actionable.
- **`Path.expand("")` / empty-string path (truthy in Elixir) mis-handled in the first `cond` branch.** Considered because only `nil`/`false` are falsy. Ruled out as a defect: `""` is a legitimately truthy value and `Path.expand("")` resolves to the cwd, which is a coherent (if unusual) dest; no crash or wrong-shape result, and no documented contract promises rejection of `""`.

