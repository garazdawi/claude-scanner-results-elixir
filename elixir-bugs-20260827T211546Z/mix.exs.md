# /Users/lukas/code/claude-scanner/elixir/lib/elixir/mix.exs
<!-- sha256: 80564a2db2b7b1b64b4570761d569cae68a2d900674659b524915ebdb60192f4 -->
<!-- tree: 3389f8405 2026-08-25 -->

# /Users/lukas/code/claude-scanner/elixir/lib/elixir/mix.exs
<!-- sha256: 80564a2db2b7b1b64b4570761d569cae68a2d900674659b524915ebdb60192f4 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Re-reviewed on 2026-08-28_

## Summary
This entry claims to review the Elixir project's bootstrap `mix.exs`. On re-review I established that **the file itself is not present in this checkout**: the review path `/Users/lukas/code/claude-scanner/elixir/lib/elixir/src/mix.exs` does not exist, and the real file at `/Users/lukas/code/claude-scanner/elixir/lib/elixir/mix.exs` (one level above the `src/` folder actually mounted) is likewise absent. The only thing mounted for this folder is `src/` = the `.erl` compiler front-end (`/cscan/0`). I confirmed absence by `stat -L`, `find`, and `File.exists?/1` (both paths → `false`).

Consequently, the prior pass's file-load evidence ("loaded the actual file via `Code.eval_file/1` … inspected `Mix.Project.config()`") **cannot be reproduced in this sandbox** — there is no file to load. I re-ran only the checks that do not depend on the file's contents (they hold), and I downgrade every ruled-out entry whose `verified` basis required loading the absent file back to `asserted`. No security findings; no positive defects. This is a downgrade-only re-review, which is the safe direction.

## Bugfixes

No findings

### Ruled out

- **`defmodule Elixir.MixProject` double-prefix / wrong module name** — In Elixir the `Elixir.` prefix *is* the root alias namespace, so `defmodule Elixir.MixProject` compiles to the atom `Elixir.MixProject`, identical to `defmodule MixProject` (not `Elixir.Elixir.MixProject`). Basis: `verified` for the **language fact** — re-ran `Elixir.MixProject == MixProject` → `true` this pass (file-independent; does not require the source). Confidence: high. Axes examined: alias-namespace resolution, atom identity | Not examined: file-content (that the absent file actually spells the module this way — the file is not in this checkout, so its text cannot be confirmed).

- **Missing `project/0` callback required by `use Mix.Project`** — The prior pass rated this `verified` on the strength of "pushed this module through a started Mix." I cannot reproduce that here because the file is absent, so no module can be loaded from it. Basis: `asserted` (**downgraded from `verified`: prior executed evidence is not reproducible — target file not present in this sandbox**). Confidence: low. Axes examined: none against the actual file | Not examined: behaviour-callback presence, resolved-config return (both require loading the file, which does not exist here).

- **`version: System.version()` mismatch / crash** — `System.version/0` is a documented public function returning the running Elixir version binary. Basis: `verified` for the **language fact** — re-ran `is_binary(System.version())` → `true` and `System.version()` → `"1.21.0-dev"` this pass (file-independent). The prior claim that "the resolved config carried `version: "1.21.0-dev"`" required loading the file and is **not** reproduced. Confidence: high on the language fact / low on the config-carried claim. Axes examined: `System.version/0` return type | Not examined: acceptance of that value by Mix as `:version` for this file (requires loading the absent file).

- **Absent `:deps` / `:application` / `:elixirc_paths` keys** — The prior pass rated this `verified` ("loaded the file into a live Mix and inspected `Mix.Project.config()`; Mix injected `deps: []` and `elixirc_paths: ["lib"]`"). That evidence cannot be reproduced — the file is not in this checkout, so `Mix.Project.config()` cannot be evaluated against it. Basis: `asserted` (**downgraded from `verified`: the executed evidence the prior pass relied on is unavailable because the target file is absent**). Confidence: low. Axes examined: none against the actual file | Not examined: default-injection for missing keys, config-merge outcome (both require loading the file).

## Did I verify this
Partially — and I am explicit about the boundary this time.

- **What I reproduced (file-independent, this pass, against live Elixir 1.21.0-dev / OTP 29):** `Elixir.MixProject == MixProject` → `true`; `is_binary(System.version())` → `true`; `System.version()` → `"1.21.0-dev"`; `Code.eval_file/1` exported → `true`. These are language/runtime facts that do not depend on the reviewed file's text.
- **What I could NOT reproduce:** the prior pass's core evidence — loading *this file* via `Code.eval_file/1` and reading `Mix.Project.config()`. The file does not exist in the sandbox: `File.exists?("/Users/lukas/code/claude-scanner/elixir/lib/elixir/mix.exs")` → `false` and `File.exists?(".../src/mix.exs")` → `false`; `stat -L` and `find /cscan -name mix.exs` return nothing. Only the `src/` `.erl` sources are mounted (`/cscan/0`). Reading neighbour modules does not help here: the standing-context neighbours are the `.erl` compiler front-end, none of which consume this bootstrap `mix.exs`, and reading them would not clear the untried file-load axis anyway.

- **On the first-pass hedge ("outside this file" — Mix build orchestration consuming this config):** I could not resolve it by execution, because the file needed to drive that consumption is absent. I did not upgrade it to a low-confidence security finding either: Mix consuming a static, project-authored project config at *build time* is not a Phase 1 untrusted-input sink — there is no attacker-controlled data flowing into that path, so there is no applicable sink class under which a finding would be meaningful. The hedge therefore remains genuinely unverified in this environment; I have marked the dependent ruled-out entries `asserted` rather than pretending the consumption path was checked.

## Dropped in re-review
No findings dropped (the first pass reported none). Changes made, all in the safe direction:

- **`Missing project/0 callback`** and **`Absent :deps / :application / :elixirc_paths keys`**: downgraded `verified` → `asserted`. New evidence: the reviewed file is not present in this checkout (`File.exists?` → `false` at both candidate paths; `find`/`stat` confirm), so the prior pass's file-load-and-inspect evidence cannot be reproduced. Per the re-review rules, a `verified` basis that rests on unreproducible execution must not be carried forward.
- **`version: System.version()`** and **`defmodule Elixir.MixProject`**: retained `verified` only for the file-independent language facts I re-executed this pass; the sub-claims that required loading the absent file (config-carried `:version`, and that the file's text actually spells the module `Elixir.MixProject`) are moved to `Not examined`.

No ruled-out entry was upgraded, and no ruled-out entry had its `Not examined` list narrowed except where an axis was genuinely re-executed without the file.
