# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.install.ex
<!-- sha256: 10a3f8da47059eb866f771cb4235333d765cc8f11a8baad1bd1bb1c435617f93 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/escript.install.ex
<!-- sha256: 10a3f8da47059eb866f771cb4235333d765cc8f11a8baad1bd1bb1c435617f93 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary

`Mix.Tasks.Escript.Install` implements the `Mix.Local.Installer` behaviour to install a prebuilt or freshly-built escript into `~/.mix/escripts`. In this re-review I resolved the two hedges the first-pass reviewer flagged ("outside this file") by reading the two callgraph neighbours they leaned on — `Mix.Local.Installer` (`local/installer.ex`) and `Mix.Tasks.Escript.Build` (`tasks/escript.build.ex`) — and re-running the `escript?/1` split logic. Both hedged assumptions are now verified against source. I still find no genuine reachable bug. I did surface one new low-severity behavioural edge (custom `shebang`/`comment`/`emu_args` build config) that I record as a low-confidence finding rather than leave as a silent assumption.

## Bugfixes

No findings

## Risks / caveats

- **Verified (was hedged): escript header is exactly three lines.** `escript.build.ex:207-216` builds the artifact as `IO.iodata_to_binary([shebang, comment, emu_args, zip])`. Reading the three producers confirms each contributes exactly one `\n`-terminated line: default `shebang = "#! /usr/bin/env escript\n"` (`escript.build.ex:209`), `build_comment/1` → `"%% #{user_comment}\n"` (`escript.build.ex:355-357`), `build_emu_args/2` → `"%%! -escript main #{escript_mod} #{user_args}\n"` (`escript.build.ex:359-361`), followed by the raw `:zip.create/3` binary. So a standard mix-built escript splits under `String.split(binary, "\n", parts: 4)` to `["#!...", "%% ...", "%%! ...", <<80,75,3,4,...>>]`, exactly what `escript?/1` (`escript.install.ex:192-195`) matches. This is now confirmed by reading the neighbour, not assumed.

- **Verified (was hedged): the in-file logic reachable after a network fetch.** Reading `local/installer.ex` shows every path (`:fetcher`, `{:local|:url, src}`, `:project`) converges on `local_install/3` (`installer.ex:127-178`), which reads bytes via `Mix.Utils.read_path/2` and then calls `module.install(basename, binary, previous_files)` (`installer.ex:132-134`). For the fetch/build paths the argument passed to `local_install` is the built path returned by `build/2` (`installer.ex:94,101`), which for this module is `Mix.Local.name_for(:escripts, Mix.Project.config())` (`escript.install.ex:137`). So `install/3` and `escript?/1` — the only logic in *this* file reachable after fetch — always receive a binary read from a real path, exactly the inputs I exercised directly. The fetch orchestration, atom-table growth from `String.to_unsafe_atom` (`installer.ex:255,259,286`), and the `try/after` temp-dir cleanup (`installer.ex:359-408`) all live in `installer.ex` and are that file's review surface, not this one.

- `check_discoverability/3` (`escript.install.ex:159-190`) produces only shell warnings; even if its `cond` ordering were arguable, it cannot corrupt state or crash, so I do not treat its branch ordering as a bug.

### New low-confidence finding

- **`escript?/1` false-rejects a self-built escript with custom `shebang`/`comment`/`emu_args` config — Confidence: low.** `escript.build.ex` lets a project override `escript_opts[:shebang]` (`escript.build.ex:209`) and interpolates `user_comment`/`user_args` verbatim into the comment/emu_args lines (`escript.build.ex:356,360`). If a project sets `shebang:` to a string not starting with `#!` (or not ending in `\n`), or embeds a `\n` inside its comment/emu_args, the resulting valid escript splits to a different header shape and `escript?/1` returns `false`, so `install/3` aborts with `"The given path does not point to an escript"` (`escript.install.ex:129`). This is a usability/robustness edge (a legitimately-built artifact is refused), not memory-unsafety or state corruption, and it overlaps the documented "does not point to an escript" contract — hence low confidence. Sink class: input-handling / documented-return-shape (`INPUT_INVALID_RETURNS_ERROR` — here it errors on *valid* input rather than crashing, so it degrades gracefully). Flagged rather than left as the first pass's silent "Erlang-native escripts are legitimately rejected" assumption.

### Ruled out

- **`escript?/1` crashing or misclassifying on binary/invalid-UTF-8 input** — `verified`. Re-ran the exact `String.split(binary, "\n", parts: 4)` + `match?/2` logic on a synthetic escript whose ZIP body contained `0x0A`, `0xFF`, and `0xC8` bytes: standard 3-line escript → `true`, no crash on invalid UTF-8 (`String.split/3` matches the `"\n"` pattern byte-wise and does not validate UTF-8). `parts: 4` caps the split before extra ZIP-body newlines are reached, so the trailing bytes are inert. Axes examined: byte-value (0x0A/0xFF/0xC8 in body), header-line-count (3), UTF-8 validity | Not examined: none.

- **`escript?/1` rejecting a valid mix-built escript** — `verified`. The 3-line header from `escript.build.ex` (confirmed above by reading `escript.build.ex:207-216,355-361`) splits to `["#!...", "%% ...", "%%! ...", <<PK...>>]`, which `match?/2` accepts (`true` in the re-run). The only inputs that reach the `else` branch are non-standard headers (custom-config or Erlang-native escripts), captured as the low-confidence finding above. Axes examined: default-build header shape (via source + run) | Not examined: none.

- **Callback contract mismatch (`@impl`/return shapes)** — `verified` (was `asserted`). Read the behaviour: `Mix.Local.Installer` declares `check_install_spec/2 :: :ok | {:error, String.t()}` (`installer.ex:46`), `find_previous_versions/1 :: [Path.t()]` (`installer.ex:51`), `build/2 :: Path.t()` (`installer.ex:57`), `install/3 :: :ok` (`installer.ex:62`). All four implementations return matching shapes: `check_install_spec/2` → `:ok` (`escript.install.ex:97-101`); `find_previous_versions/1` → `[dst]` or `[]` (`escript.install.ex:104-107`); `install/3` → `:ok` or `Mix.raise` (`escript.install.ex:110-131`); `build/2` → `Mix.Local.name_for/2`, a string (`escript.install.ex:134-138`). `run/1` satisfies the `Mix.Task` behaviour. No spec/impl divergence.

- **`install/3` resource leak on error path** — `verified`. No long-lived resources are opened: files are written via bang calls (`File.mkdir_p!`/`File.write!`/`File.chmod!`, `escript.install.ex:120-123`) that raise before any handle is retained; no ports/ETS/processes. The `_ = File.rm(dst)` / `_ = File.rm(dst <> ".bat")` (`escript.install.ex:114-115`) intentionally ignore "file absent" errors, which is correct for pre-deletion. No `try/after` is needed because nothing acquires a releasable resource. Axes examined: file handles, ports, ETS, processes | Not examined: none.

- **`build/2` returning wrong type** — `verified`. Returns `Mix.Local.name_for(:escripts, Mix.Project.config())`, a `Path.t()` string, matching the `build/2 :: Path.t()` callback (`installer.ex:57`).

## Did I verify this?

- Read the target file in full (`escript.install.ex:1-196`).
- Read the flagged neighbour `local/installer.ex:1-436` to confirm the post-fetch call path into `install/3`/`escript?/1` — the basis of the first hedge — is exactly the in-file logic I exercised directly.
- Read `tasks/escript.build.ex:207-224,345-361` to confirm the three-line header shape — the basis of the second hedge — directly from the producing code.
- Re-ran the `escript?/1` split+match logic on a synthetic escript with 0x0A/0xFF/0xC8 bytes in the ZIP body; standard escript → `true`, no crash. The Erlang toolchain in the sandbox executed this.
- I did **not** execute the git/github/hex fetch end-to-end (network); that orchestration and its `String.to_unsafe_atom` atom-table and temp-dir cleanup concerns belong to `installer.ex`'s own review, and the only *this-file* code reachable afterward (`install/3`, `escript?/1`) was verified directly.

### Dropped in re-review

None. Every prior finding/ruled-out entry is preserved; the two prior "Risks / caveats" hedges were upgraded to verified statements (with the neighbour source I read), and one previously-silent assumption ("Erlang-native escripts are legitimately rejected") was upgraded to an explicit low-confidence finding.
