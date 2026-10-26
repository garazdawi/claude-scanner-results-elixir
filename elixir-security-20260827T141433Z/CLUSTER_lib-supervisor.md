I've read both cluster files and traced how each is actually wired into the rest of the library.

**What the two files do and how they connect:**

- `default.ex` — `Supervisor.Default.init(args) -> args` is a pure pass-through callback. Its `init_arg` is *always* constructed internally by trusted library code: `Supervisor.start_link/2` builds it via `Supervisor.init/2` (`supervisor.ex:737`, returning `{:ok, {flags, normalized_children}}`) and `DynamicSupervisor.start_link/1` likewise (`dynamic_supervisor.ex:305`). Validation/normalization happens in `init/2`'s `init_child` (`supervisor.ex:816-854`), never in `Default`.

- `spec.ex` — `Supervisor.Spec.supervise/2` / `worker/3` / `supervisor/3` build the deprecated 6-tuple format with tuple-shaped flags `{{strategy, maxR, maxS}, children}`. This is returned *directly* from a module-based supervisor's `init/1` to Erlang `:supervisor`; it is imported only via `use Supervisor` (`supervisor.ex:529`).

**Why there is no cross-file finding between them:**

The two files sit on entirely separate code paths and never exchange data. `Supervisor.Default` handles the modern `init/2`-constructed arg; `Supervisor.Spec` builds the legacy tuple returned directly to `:supervisor` — the legacy path does not flow through `Supervisor.Default`, and `Supervisor.Default`'s arg is never produced by `Supervisor.Spec`. There is no shared sink reached by two paths, no validation performed in one and skipped by the other, and no common security-relevant invariant split between them.

Against the security lens specifically: both files consume developer-authored supervision specs (code, i.e. trusted by definition — a caller supplying a malicious child spec already has code execution), so there is no untrusted-input trust boundary; there is no taint flow, no authn/authz split, no crypto, and (being BEAM/GC'd Elixir) no memory-safety coupling. `assert_unique_ids`'s O(n²) scan and `Default`'s zero-validation pass-through are each single-file observations on trusted, non-attacker-reachable input — not cross-file issues, and out of scope for this lens.

No cluster cross-file findings
