# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/file/stat.ex
<!-- sha256: fb58203093233e9fe04ccdeaf89610166d4e6cd95c1fceb4b5044d01bfb82508 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:03:33Z_

## Summary

`File.Stat` is a small, macro-generated record⇄struct converter that round-trips correctly; I found no reachable bugs through its documented public API (`to_record/1`, `from_record/1`).

## Bugfixes

No findings

## Risks / caveats

- **`@type` says `:undefined`, `defstruct` defaults to `nil`.** The `@type t()` (stat.ex:66–80) allows each field to be `... | :undefined`, but `defstruct keys` (stat.ex:64) gives every field a default of `nil`, so `%File.Stat{}` — and any partially-populated struct — is a value the `@type` does not describe, and `to_record(%File.Stat{})` yields `{:file_info, nil, nil, …}` where `:file.file_info()` expects `:undefined`/integers. This is a Dialyzer-visible contract inconsistency rather than a runtime defect, and it is identical to canonical upstream Elixir; in real usage these structs are produced by `File.stat/2` (fully populated) or hand-built by callers who supply values, so the mismatch is effectively never hit. I did not raise it as a finding because it is not reachable as a wrong result or crash through the module's documented API — every field the public functions touch is threaded positionally and verified below.
- **Verification scope.** I verified the field ordering and round-trip behaviorally: `elixir -e 'r = File.Stat.to_record(%File.Stat{size: 10, type: :regular}); File.Stat.from_record(r)'` reproduced the struct exactly, and the emitted tuple `{:file_info, 10, :regular, nil…}` is a 14-element `:file_info` record with the 13 fields in Erlang record order. The `keys`/`vals`/`pairs` macro plumbing (stat.ex:59–62) derives ordering directly from `Record.extract(:file_info, …)`, so struct-field order and record-tuple order cannot drift apart — the class of "state/accumulator threading" or "pattern match on wrong shape" bug is structurally excluded here.

### Ruled out
- **Field-order mismatch between `to_record` and `from_record`** — both use the same `vals`/`pairs` lists generated once from `Record.extract`, so a reordering in one without the other is impossible. Basis: `verified` (round-trip produced identical struct).
- **Missing/extra field vs the `:file_info` record** — `defstruct keys` and the tuple are both generated from the extracted record, so they cannot desynchronize from the actual `file.hrl` layout at compile time. Basis: `verified` (struct exposed exactly the 13 documented fields; tuple was 14-element).
- **`nil` default causing a crash in `to_record`** — `to_record` only pattern-matches the struct and rebuilds a tuple; it performs no arithmetic or type-sensitive operation on the values, so `nil` fields pass through without raising. Basis: `verified` (`to_record(%File.Stat{})`-style call produced `{:file_info, …, nil}` without error).

