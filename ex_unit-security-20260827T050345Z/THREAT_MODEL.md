```markdown
# Threat Model: `elixir/lib/ex_unit` (ExUnit test framework)

## Protocol / standard implemented
This is **ExUnit**, Elixir's built-in unit-testing framework — *not* a network
protocol implementation. There is no RFC/NIST spec governing it; its "contract"
is Elixir's own documented public API and the `mix test` CLI conventions.
This is **developer tooling that executes trusted test code in-process on the
BEAM**. There is no network surface and no untrusted peer in the normal model.

Documented public entry points (the modules a caller `use`s or calls):
- `ExUnit` — `start/1`, `configure/1`, `run/0` (suite lifecycle).
- `ExUnit.Case` / `ExUnit.CaseTemplate` — `use`-macros defining test modules.
- `ExUnit.Callbacks` — `setup`, `setup_all`, `on_exit`, `start_supervised`.
- `ExUnit.Assertions` — `assert/refute` and friends (macro-expanded).
- `ExUnit.DocTest` — `doctest/1`; extracts & runs code from `@doc`/`@moduledoc`.
- `ExUnit.Filters` — `parse/1`, `parse_paths/1`, `eval/4` (CLI arg parsing).
- `ExUnit.CaptureIO` / `CaptureLog` — group-leader / logger interception.
- `ExUnit.Formatter` / `CLIFormatter` — output rendering.

## Trust / contract boundaries
Almost every input to this folder originates from a **trusted local developer**:
the test source, the doctests in docstrings, the module-under-test, and the CLI
arguments. The reviewer should *not* apply a network-attacker model here. The
boundaries that do exist:
- **CLI / operator config** → `ExUnit.Filters.parse/1` and `extract_location/1`:
  `foo:bar`, `line:9`, `path:line` tokens. Note `parse/1` uses
  `String.to_unsafe_atom/1` on the key — unbounded atom creation, but the source
  is the developer's own command line (self-inflicted, not a privilege crossing).
- **On-disk file** → `ExUnit.FailuresManifest.read/1` / `info/1` reads
  `.mix/.../*.manifest` via `:erlang.binary_to_term/1`. This is **not** called
  with the `:safe` option; `safe_binary_to_term/1` only `rescue`s `ArgumentError`
  around a plain `binary_to_term`. If an attacker can plant a manifest file, this
  is a classic term-deserialization sink (atom/resource exhaustion). Trust here
  rests entirely on the assumption that `.elixir_ls`/`_build` is not attacker-writable.
- **Code under test** → `DocTest` compiles and `Code.eval`s expressions pulled
  from docstrings, and assertions capture/re-raise arbitrary terms. All treated
  as trusted developer code.

## Attacker model
In scope per ExUnit's implicit model: **nobody hostile** — it defends against
*accidental* misbehaviour (a test crashing, an `on_exit` leaking, a runaway
async case) so that one bad test doesn't corrupt the run of its siblings.
Out of scope: network peers (none), and a malicious operator (the operator *is*
the test author, who already has arbitrary code execution). The only genuinely
adversarial surfaces worth flagging are the **manifest deserialization** and
**atom-table growth from filters** — reachable only by a co-located user who can
write build artifacts or inject CLI args, i.e. someone who likely already has
code-exec. Weight findings accordingly (Low unless a concrete privilege crossing exists).

## Common attack patterns for this class
Not a crypto/protocol library, so the usual CVE families don't map. The relevant
cues for a test-runner:
- **Unsafe `binary_to_term`** (manifest) → atom exhaustion / resource DoS.
- **`String.to_atom`/`to_unsafe_atom` on external input** → atom-table DoS.
- **Isolation failures**: an `on_exit` handler, `setup_all`, or captured
  group-leader that leaks into or crashes a sibling test/process.
- **Signal handling** (`trap_signal(:sigquit …)` in `Runner`) — trap/untrap must
  be balanced under `try/after`; a leaked trap corrupts later runs.
- **Format-string / inspect-time crashes** in the formatter on pathological terms.

## Supervisor / process structure
- `ExUnit.Server` (`GenServer`, named) — owns the module queue and the shared
  public ETS table used by `OnExitHandler`. **Single point of failure**: a crash
  here kills the whole suite and the ETS table other processes depend on.
- `ExUnit.OnExitHandler.Supervisor` (`Supervisor`, `:one_for_one`,
  `max_restarts: 1_000_000 / max_seconds: 1`) — deliberately near-infinite
  restart intensity so a flapping on-exit child never trips the supervisor. It
  propagates `$callers` into the child so ownership/monitor semantics survive.
- `ExUnit.CaptureServer` (`GenServer`) — serializes global device/logger capture;
  a crash here breaks output interception for concurrent async tests.
- `ExUnit.EventManager` — fan-out to formatter/stats handlers.
- `ExUnit.Runner` — the driver process; spawns one linked process per test
  module, bounded by `max_cases`; monitors and reaps them so a test crash is
  reported, not propagated.

## Cross-file invariants (what a per-file reviewer would miss)
- **`Filters.eval/4` gates execution**: `Runner` relies on `eval/4` having
  already excluded/skipped tests; `has_tag/3` line-matching assumes the *whole*
  `collection` is passed so `closest_test_before_line/2` is meaningful — a caller
  passing a partial collection silently mis-selects tests.
- **`Server` owns the ETS table** named `ExUnit.Server` that `OnExitHandler`
  reads/writes; `OnExitHandler` assumes `Server.init/1` created it `:public`
  before any handler runs. Don't rule on ETS access in `OnExitHandler` without
  this.
- **`FailuresManifest.put_test/2` is the sole writer**; it drops `:skipped`/
  `:excluded` and deletes on `nil` state, so `Filters.failure_info/1` consumers
  assume the on-disk set contains only genuinely failed/invalid IDs.
- **Runner balances `trap_signal`/`untrap_signal`** via `try/after`; reviewers
  seeing the trap installed in isolation should confirm the `after` clause is the
  guarantee it's removed.
- **`max_restarts: 1_000_000`** on the OnExit supervisor is intentional, not a
  bug — it exists so operator/test cleanup failures never cascade.
```