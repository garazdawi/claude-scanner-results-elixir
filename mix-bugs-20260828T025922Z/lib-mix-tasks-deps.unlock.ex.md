# /Users/lukas/code/claude-scanner/elixir/lib/mix/lib/mix/tasks/deps.unlock.ex
<!-- sha256: 4d073bbaf18cd49b95b5e190d198a7d97827848095d45f8ac621dfb782b7ad34 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T04:50:44Z_

## Summary

`deps.unlock.ex` is a small, well-formed Mix task; its only defect is a cosmetic mishandling of duplicate dependency arguments arising from list-subtraction (`--`) semantics.

## Bugfixes

#### `CSCAN-deps.unlock-duplicate-arg-spurious-warning`

- **Finding ID:** `CSCAN-deps.unlock-duplicate-arg-spurious-warning`
- **Severity:** Low
- **Confidence:** high
- **Location:** `deps.unlock.ex:74-83`, `do_run/2` (the `apps != []` branch)
- **Class:** Off-by-one / wrong arithmetic (set/multiset semantics of `--`)
- **Impact:** Running `mix deps.unlock foo foo` (the same locked dependency named more than once) prints a false `warning: foo dependency is not locked` even though `foo` is locked. The dependency is still unlocked correctly, so the effect is a misleading warning, not a wrong action.
- **Evidence:** `verified`. The `--` operator removes each right-hand element only once from the left. With `apps = [:foo, :foo]` and `Map.keys(lock) = [:foo]`:
  ```
  $ elixir -e 'apps = [:foo, :foo]; keys = [:foo]; unlocked = apps -- keys; IO.inspect(unlocked, label: "unlocked(warned as not locked)"); IO.inspect(apps -- unlocked, label: "will unlock")'
  unlocked(warned as not locked): [:foo]
  will unlock: [:foo]
  ```
  `unlocked` (the "not locked" set) is non-empty, so the `for app <- unlocked` loop at line 79 emits the spurious warning for a dependency that is in fact locked.
- **Suggested fix:** Deduplicate the parsed apps before the subtraction, e.g. `apps = apps |> Enum.map(&String.to_unsafe_atom/1) |> Enum.uniq()`.

## Risks / caveats

- I could not drive the full `mix deps.unlock` task end-to-end (the sandbox `/work` is not a Mix project), so the branch-selection `cond` and `Mix.Dep.Lock.read/write` interactions were reviewed by reading plus an isolated test of the `--` operator that produces the bug. The list-subtraction behaviour itself is verified.
- `String.to_unsafe_atom/1` on argument names (line 76) creates atoms from operator-supplied strings; I treat this as intended (developer-invoked CLI, per the folder threat model) rather than a bug, but it is worth noting for atom-table growth if this task is ever driven programmatically with untrusted names.

### Ruled out

- **`filter = opts[:filter]` truthiness in `cond` (line 64):** `--filter` is declared `:string`, so `opts[:filter]` is always a binary when present and `nil` when absent; `nil` is falsy and skips the branch correctly. `=~` with a string RHS is substring matching (not regex), so no regex-compile crash. No defect.
- **Empty-string `--filter ""`:** an empty pattern makes `=~ ""` match every app, unlocking all — but this requires the operator to explicitly pass an empty filter, a degenerate self-inflicted input, not a reachable defect on well-formed use.
- **`Enum.join(apps, "\n* ")` on atoms (line 110):** atoms implement `String.Chars`, so joining works; verified mentally against `to_string` semantics — no crash.
- **`String.to_unsafe_atom/1` atom creation (line 76):** deliberate for a developer-invoked CLI task per the folder threat model; not a reachable network/untrusted-input path.

