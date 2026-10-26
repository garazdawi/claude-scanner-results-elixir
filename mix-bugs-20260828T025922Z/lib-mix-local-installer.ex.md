# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/local/installer.ex
<!-- sha256: d1ffabbbeb90dcfc56fc550707a9f6d96d78c75090c5c738fe5b73bf930bc8ab -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T03:18:29Z_

## Summary

`Mix.Local.Installer` implements shared archive/escript install/uninstall/fetch logic; the code is largely sound, but its URL handling in `local_install/3` crashes with an opaque error when a `:url` install spec has no path component.

## Bugfixes

#### `CSCAN-installer-url-nil-path-basename`
- **Finding ID:** CSCAN-installer-url-nil-path-basename
- **Severity:** Medium
- **Confidence:** high
- **Location:** `lib/mix/lib/mix/local/installer.ex:128`, `local_install/3`
- **Class:** Missing clause / non-total function (nil not handled)
- **Impact:** Running `mix escript.install <url>` where the URL has no path component (e.g. `http://example.com`, a bare host, or a host with only a query/fragment) crashes with a `FunctionClauseError` deep inside `IO.chardata_to_string/1` instead of either fetching the artifact or emitting a clear error. The message tells the caller nothing about what is wrong with their input.
- **Evidence:** `verified`.

`URI.parse/1` returns a `nil` path for a host-only URL, and `Path.basename(nil)` raises:

```
$ elixir -e 'IO.inspect(URI.parse("http://example.com").path)'
nil

$ elixir -e 'IO.inspect(Path.basename(URI.parse("http://example.com").path))'
** (FunctionClauseError) no function clause matching in IO.chardata_to_string/1
    The following arguments were given to IO.chardata_to_string/1:
        # 1
        nil
```

Reachability trace through the documented public API `install/3`:
- `parse_args(["http://example.com"], opts)` → single-arg clause (line 210): `local_path?` false, `file_url?` true (scheme `http`) → `{:url, "http://example.com"}`.
- `Mix.Tasks.Escript.Install.check_install_spec({:url, _}, _)` returns `:ok` for any URL (`lib/mix/lib/mix/tasks/escript.install.ex:96`), so no extension guard rejects it (archive.install *does* reject via `check_extname`, but escript does not).
- `install/3` enters the `{path_or_url, src} when path_or_url in [:local, :url]` branch → `local_install(module, "http://example.com", opts)`.
- Line 128 `basename = Path.basename(URI.parse(src).path)` computes `Path.basename(nil)` → crash, before any prompt or download.

- **Suggested fix:** Guard against a nil path when deriving the basename, e.g. `basename = Path.basename(URI.parse(src).path || src)`, or detect the nil path and `Mix.raise` a message pointing at the malformed URL (no filename to install to).

## Risks / caveats
- I verified the crash mechanism (`Path.basename(nil)`) and traced reachability via `escript.install`'s permissive `check_install_spec`, but did not execute a full end-to-end `mix escript.install http://example.com` invocation (would require network/interactive shell); the pre-download crash at line 128 makes network irrelevant.
- I reviewed all documented public entry points (`install/3`, `parse_args/2`, `uninstall/4`, `fetch/3`). The `String.to_unsafe_atom` uses on CLI package/app names are the noted intentional design (operator-controlled input), not treated as a new defect.
- `uninstall/4` uses `OptionParser.parse!(argv, switches: switches)` (lenient) while `install/3` uses `strict:`; this asymmetry is plausibly intentional and I could not establish a behavioural bug from it, so it is not reported.

### Ruled out
- **`parse_args(["github" | rest])` MatchError on `["github"]`** — the bare single-element list `["github"]` is matched by the earlier single-arg clause (line 210) and returned as an `{:error, ...}` local-path message; it never reaches the `["github" | rest]` clause, so `[repo | rest] = []` is not triggered. Verified by clause-ordering: line 210 precedes line 218 and matches one-element lists exclusively.
- **`fetch/3` middle default argument** — `def fetch(dep_spec, in_fetcher \\ &in_fetcher/1, in_package)` is a legal Elixir middle-default (trailing arg required); `fetch/2` binds the second arg to `in_package` with the default fetcher, which matches the intended two-arity call sites.
- **`install/3` case exhaustiveness (line 85)** — the three branches cover every `install_spec` shape (`{:fetcher, _}`, `{:local|:url, _}`, `:project`); `{:error, _}` is already converted to `Mix.raise` above, so no `CaseClauseError` is reachable.

