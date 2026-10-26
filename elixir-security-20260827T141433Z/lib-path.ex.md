# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/path.ex
<!-- sha256: b6144de2c362f1167f29efc86511520e6a29d2221736bd74df0d88f3a3f10649 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:17:45Z_

I have enough verified evidence. `IO.chardata_to_string` on a binary is O(1), confirming the super-linear cost in `join/1` comes from re-copying the growing accumulator via `<>` at each step. Writing up the report.

---

## Summary

`Path`'s confinement primitives (`safe_relative/2`, `safe_join/2`) correctly reject `..`, symlink, and absolute-path escapes, but `join/1` has verified super-linear cost on large caller-supplied lists, and the safe primitives pass embedded NUL bytes through where the sibling `wildcard/2` rejects them.

## Security

#### `CSCAN-path-join-quadratic`
- **Finding ID:** `CSCAN-path-join-quadratic`
- **Severity:** Low
- **Confidence:** high
- **Location:** `path.ex:646-648` (`join/1`); cost sink is the `<>` in `do_join/3` at `path.ex:687`.
- **Class:** Resource consumption
- **Issue:** `join/1` folds a list by repeatedly calling `join/2`, and each `join/2` rebuilds the whole accumulated left side (`remove_dir_sep(left) <> "/" <> relative(right)`). Because `IO.chardata_to_string/1` on a binary is O(1) (verified below), the dominant cost is the `<>` concatenation copying the *growing* prefix binary at every one of N steps, producing ≥ quadratic time and heavy garbage generation in the number of list components.
- **Impact / threat:** A caller that builds a path by handing `Path.join/1` a large, externally-influenced list of segments (request path segments, archive entry parts, a parsed manifest) can be driven into multi-second, single-scheduler CPU plus GC pressure — ~3 s for 80k components, with input-size→CPU amplification that is super-linear. That blocks a scheduler and enables a denial-of-service against the process doing the join. Trust boundary: data forwarded by the caller from the network/disk/an archive into `join/1`.
- **Evidence:** `verified`.
  - `Path.join(List.duplicate("abcdefgh", n))`: `n=20000 → 32 ms`, `40000 → 447 ms`, `80000 → 2928 ms` (doubling input ≫ 4× time).
  - `for _ <- 1..1000, do: IO.chardata_to_string(big_5MB_binary)` → `0 ms`, confirming the per-step conversion is O(1) and the quadratic is the growing-prefix `<>` copy.
- **Suggested fix:** Accumulate the joined result as iodata and call `IO.iodata_to_binary/1` once at the end, so the growing prefix is never re-copied per element (turn the O(N²) prefix-copy into O(N)).

#### `CSCAN-path-safe-relative-nul-byte`
- **Finding ID:** `CSCAN-path-safe-relative-nul-byte`
- **Severity:** Low
- **Confidence:** medium
- **Location:** `path.ex:941-948` (`safe_relative/2`); reached transitively by `safe_join/2` (`path.ex:722-726`) and the deprecated `safe_relative_to/2` (`path.ex:902-904`).
- **Class:** File operations and path handling
- **Issue:** `safe_relative/2` returns `{:ok, path}` for input containing an embedded NUL byte, passing it through unchanged (`Path.safe_relative("a\0b")` → `{:ok, <<97, 0, 98>>}`). The sibling `wildcard/2` explicitly rejects NUL (`chardata_to_list!/1`, `path.ex:842-858`) precisely because it is dangerous in a path, so the handling is inconsistent within the module even though `safe_relative` is documented as "protected from directory-traversal attacks."
- **Impact / threat:** The documented confinement contract (no `..`/symlink escape above root) is *not* broken by a NUL, and BEAM file operations reject NUL paths with `:badarg` (verified) — so there is no in-BEAM path-truncation traversal. Residual risk exists only when a caller forwards the "sanitized" path to a sink that is *not* a BEAM file op and that treats NUL as a terminator (an OS-command argv it builds by hand, a port/driver/NIF, or serialization into another format). This is a defense-in-depth gap and an internal inconsistency, not a break of the stated contract. Trust boundary: an untrusted path (upload name, archive entry, manifest field) — exactly the intended input to this sanitizer.
- **Evidence:** `verified`. `Path.safe_relative("a\0b")` → `{:ok, <<97,0,98>>}`; `File.read(<<..0..>>)` → `{:error, :badarg}` (BEAM refuses, no truncation); `Path.wildcard("foo\0*")` → `ArgumentError "…null byte…"`.
- **Suggested fix:** Reject NUL in `safe_relative/2` (return `:error` when the string contains `0`), mirroring the existing null-byte guard in `wildcard/2`.

### Ruled out

- **Non-confining `join/2`, `expand/1,2`, `absname/1,2`, `relative_to/3` used for traversal** — documented as *not* confining, with `safe_relative`/`safe_join` provided as the confinement primitives. Verified those primitives reject real traversal (`../bar`, `foo/../../bar`, `a/b/c/../../../../etc`, `/etc/passwd` → `:error`), symlink escapes (`link→/etc` → `:error`), and correctly treat `%2f` and (on this OS) backslash as literals rather than separators. Basis: `verified`.
- **`wildcard/2` with an untrusted glob** — `**` walk cost is proportional to the filesystem and is the function's documented purpose (a caller decision, no code defect here); brace expansion does *not* blow up combinatorially (`{a,b}×16` completed in <1 ms), and NUL is rejected. Basis: `verified`.
- **NUL byte through `join`/`expand`** — passes through as a plain byte, but BEAM file operations reject NUL paths with `:badarg` (no truncation), and these functions are documented pure string manipulation. Basis: `verified`.
- **Symlink TOCTOU in `safe_relative`** — the symlink check races any later use of the path; this is inherent to every path-string (non-fd) confinement check and is documented behavior, not specific to this file. Basis: `asserted` (design-inherent).
- **Code-exec / atom-creation / command-exec / deserialization sinks** — none present: the file contains no `Code.*`, `*.to_atom`, `Module.concat`, `System.cmd`/`:os.cmd`, or `binary_to_term`; the only I/O is `:filelib.wildcard` (no shell). Basis: `verified` (full-file read).

## Risks / caveats

- **Windows-only paths not exercised.** The sandbox is Linux, so `major_os_type()` is `:unix` and the `:volumerelative` branch (`absname_vr/3`, and the long-list `absname_join/2` it feeds) is unreachable here. Structurally, `absname_join/2` has the same re-copy-the-growing-prefix shape as `join/1` and would likely be similarly super-linear for Windows volume-relative paths with many components, but I could not reproduce it on this platform — treat that as an unverified observation, not a ruling. The win32 `pathtype`/`split_absolute?` clauses were likewise reviewed by reading only.
- **`safe_*` correctness depends on `:filelib.safe_relative_path/2`** (OTP), which does the actual `..`/symlink adjudication; I validated its outputs behaviorally through this file's public API but did not audit the OTP implementation itself.
- The `join/1` severity is Low because it requires a large, attacker-influenced *list* reaching `Path.join/1`; the amplification is real but the input shape is situational.

