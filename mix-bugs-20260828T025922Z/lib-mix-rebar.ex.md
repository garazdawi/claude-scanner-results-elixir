# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/rebar.ex
<!-- sha256: a71a025d83a410fced1f4679eb36a802fde701effe4ab0d3d41b7fcfce7dc766 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:24:11Z_

Based on my review, here is the report.

## Summary

`Mix.Rebar` translates rebar (rebar2/rebar3) configuration and dependency declarations into Mix's formats; the parsing is mostly sound, but the git "default branch" handling matches an empty *binary* that `:file.consult` can never produce, and one git-subdir option leaks a charlist where every sibling option is a string.

## Bugfixes

#### `CSCAN-rebar-empty-git-ref-charlist`
- **Finding ID:** CSCAN-rebar-empty-git-ref-charlist
- **Severity:** Medium
- **Confidence:** high
- **Location:** `elixir/lib/mix/lib/mix/rebar.ex:179` (`parse_source/1`, the `ref` `case`)
- **Class:** Binary vs charlist vs string (dead clause + wrong return)
- **Impact:** A legacy rebar git dependency written `{app, {git, "url", ""}}` (the rebar convention for "track the default branch / HEAD") is translated to `ref: ""` instead of `branch: "HEAD"`. Downstream the Git SCM treats `""` as a real, truthy revision (`get_opts_rev/1` returns it) and attempts to check out an empty ref, which fails — the dependency cannot be fetched. The `["" | _] -> [branch: "HEAD"]` clause is effectively dead code for any config loaded via `load_config/1`.
- **Evidence:** `verified`. Erlang's `:file.consult` renders the source string `""` as the empty **charlist** `[]`, never the empty **binary** `<<>>` the pattern `["" | _]` requires. End-to-end through the documented public API:
  ```
  $ cat /tmp/rebartest/rebar.config
  {deps, [ {legacy_dep, {git, "https://example.com/legacy.git", ""}} ]}.

  $ elixir -e 'cfg = Mix.Rebar.load_config("/tmp/rebartest"); IO.inspect(cfg); IO.inspect(Mix.Rebar.deps(cfg))'
  loaded config: [deps: [legacy_dep: {:git, ~c"https://example.com/legacy.git", []}]]
  deps: [legacy_dep: [override: true, git: "https://example.com/legacy.git", ref: ""]]
  ```
  For comparison, feeding a synthetic binary `""` (which cannot arise from `consult`) does hit the intended branch:
  ```
  $ elixir -e 'IO.inspect Mix.Rebar.deps([deps: [{:foo, {:git, ~c"https://x/y", ""}}]])'
  [foo: [override: true, git: "https://x/y", branch: "HEAD"]]
  ```
- **Suggested fix:** Match the empty charlist as well, e.g. add `[[] | _] -> [branch: "HEAD"]` (or replace `["" | _]` with a guard `[v | _] when v == "" or v == []`).

#### `CSCAN-rebar-sparse-dir-not-string`
- **Finding ID:** CSCAN-rebar-sparse-dir-not-string
- **Severity:** Low
- **Confidence:** high
- **Location:** `elixir/lib/mix/lib/mix/rebar.ex:173` (`parse_source/1`, `git_subdir` case)
- **Class:** Binary vs charlist vs string (type inconsistency)
- **Impact:** For a `git_subdir` dependency the `:sparse` option is emitted as a **charlist** (`~c"sub/dir"`), while every other option produced by `parse_source/1` (`git`/url, `branch`, `tag`, `ref`) is converted to a binary via `to_string/1`. It happens to survive current downstream use (`Path.join/2` and `File.write!/2` in `Mix.SCM.Git` accept chardata), so no crash is observed, but it is an unintended type that any binary-expecting consumer (`String.*`, exact `==` lock comparison against a hand-written binary value) would mishandle.
- **Evidence:** `verified`.
  ```
  $ elixir -e 'IO.inspect Mix.Rebar.deps([deps: [{:foo, {:git_subdir, ~c"https://github.com/foo/bar", {:branch, ~c"main"}, ~c"sub/dir"}}]])'
  [foo: [override: true, git: "https://github.com/foo/bar", branch: "main", sparse: ~c"sub/dir"]]
  ```
  `git` and `branch` are binaries; `sparse` is a charlist.
- **Suggested fix:** Convert it: `{:git_subdir, [_, sparse_dir | _]} -> {:git, [sparse: to_string(sparse_dir)]}`.

## Risks / caveats
- The whole module is `@moduledoc false`, but each function under review carries a `@doc` and `deps/1`/`load_config/1` are the documented callable surface used elsewhere in Mix; I treated those as the public API. Both findings were reproduced through `load_config/1`/`deps/1`.
- The concrete downstream cost of the empty-ref bug (git checkout of `""` failing) is reasoned from `Mix.SCM.Git` (`get_opts_rev/1`, line 322) rather than executed against a real repo; the *value* produced by `Mix.Rebar` (`ref: ""`) is verified, the git failure is asserted.
- I did **not** find a functional bug in the 2-tuple `{app, req}` path skipping `compile_req/1` (ruled out below) or in `apply_overrides/3`.

### Ruled out
- **2-tuple `{app, req}` skips `compile_req/1`** — `parse_dep({app, req}) when is_list(req)` returns `List.to_string(req)` directly rather than compiling a regex. A bare 2-tuple is a source-less (hex) dependency, whose requirement is a SemVer string, not a tag-matching regex; the regex form only applies to git deps (3/4-tuples with a source), which do call `compile_req/1`. No defect.
- **`parse_source/1` `[ref | _] -> to_string(ref)` raising on odd shapes** — only reachable by passing a malformed source tuple that violates the rebar declaration contract; contract-violating input is out of scope.
- **`local_rebar_path/1` destructuring `String.split(System.version(), ".")`** — `System.version()` always has at least `major.minor`, so `[major, minor | _]` is total for valid inputs; verified the shape via `elixir -e` returns a 3-element split.
- **`apply_overrides/3` `value ++ old_value` in the `:add` branch** — requires override values that aren't lists, which violates the documented override shape; no reachable defect with well-formed overrides.

