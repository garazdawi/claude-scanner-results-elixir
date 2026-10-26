# /Users/lukas/code/claude-scanner/elixir/lib/iex/lib/iex/introspection.ex
<!-- sha256: 35f85d16c0b95928301fe539f9d4e3d70eed35dbf7f6fc018a4ec3ffdc501f1b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T02:17:53Z_

## Summary

Exhaustive dynamic sweeps over every loaded module found no triggerable crash in this module's public helpers; the only defensible finding is an inconsistency in `t/1`, which lists private (`@typep`) type definitions that the by-name forms deliberately refuse to show.

## Bugfixes

#### `CSCAN-introspection-t1-leaks-private-types`
- **Finding ID** — `CSCAN-introspection-t1-leaks-private-types`
- **Severity** — Low
- **Confidence** — low
- **Location** — `introspection.ex:765-777`, `def t(module) when is_atom(module)`
- **Class** — Contract violation / behavioural inconsistency (information disclosure)
- **Impact** — `t(SomeModule)` prints the full definitions of `@typep` (private) types, including their internal structure, whereas the sibling forms `t({module, type})` (line 791) and `t({module, type, arity})` (line 820) both filter `kind in [:type, :opaque]` and answer `"... or <type> is private"` for the same private type. A caller listing a module's types sees implementation-detail types that are otherwise hidden; the two entry points disagree about whether private types are part of the introspectable surface.
- **Suggested fix** — In the arity-1 clause, filter the fetched types before sorting/printing, mirroring the other clauses: `types |> Enum.filter(&match?({kind, _} when kind in [:type, :opaque], &1)) |> Enum.sort_by(...)`.
- **Threat note** — Not attacker-reachable; reached through the ordinary `t/1` IEx helper on a locally trusted module. This is a consistency/disclosure defect, not a memory- or crash-safety issue.
- **Evidence** — `verified`. Ran against the built Elixir:
  ```
  $ elixir -e 'IEx.Introspection.t(:sets)'
  @typep seg() :: tuple()
  @typep segs(_element) :: tuple()
  @type set() :: set(_)
  @opaque set(element)
  @typep set_v1(element) :: record(:set, field_type(:segs, segs(element)))
  @typep set_v2(element) :: %{optional(element) => []}

  $ elixir -e 'IEx.Introspection.t({:sets, :seg})'
  No type information for :sets.seg was found or :sets.seg is private
  ```
  The `@typep seg`/`segs`/`set_v1`/`set_v2` definitions are printed by `t/1` but refused by the by-name form. (I could not confirm against upstream whether this is intended longstanding behaviour — hence low confidence — but the two entry points are demonstrably inconsistent.)

### Ruled out

- **`type_doc/5` MatchError on an undocumented public type** (line 845: `{_, _, _, content, metadata} = Enum.find(docs, ...)`). If `docs` is a non-empty list (truthy) and the requested type has no matching `{:type, name, arity}` entry, `Enum.find` returns `nil` and the destructure would raise. I verified this is **not triggerable**: for every module in `:code.all_available()`, every public (`:type`/`:opaque`) type returned by `Typespec.fetch_types/1` has a corresponding doc entry whenever a Docs chunk is present (the compiler emits `:none`-doc entries for undocumented public types; opaque entries are stored with kind `:type`, confirmed empirically). A full sweep calling `t({m, name})` and `t({m, name, arity})` for every public type of every loaded module produced zero crashes.
- **`rewrite_source/1` MatchError on a source path with no `lib`/`src` component** (lines 338-346: `{in_app, [lib_or_src | _]} = ...`). If `Enum.split_while` never hits `"lib"`/`"src"`, the second element is `[]` and `[lib_or_src | _]` cannot match, raising instead of returning the `{:error, reason}` that `source_location/1` documents. Not triggerable in practice: this path runs only for modules in `@apps` or under `:code.root_dir()`, whose recorded compile source always lives under `lib/` or `src/`. A sweep of `source_location/1` over every loaded module produced zero crashes.
- **`color_prefix_with_line/1` MatchError** (`[left, right] = :binary.split(string, " ")`). Formatted typespecs always carry an `@kind ` prefix, guaranteeing at least one space; thousands of invocations via the `t`/`h`/`b` sweeps never raised.
- **`decompose/2` sigil/arity branches** — the 2-arg `{maybe_sigil, [_, _]}` branch and `{:/, _, [call, arity]}` branch fall through to `:error` for non-matching shapes; no missing clause found.
- **`h`/`b` missing-clause or wrong-arity crashes** — sweeping `h(m)`, `b(m)`, `h({m,f})`, `h({m,f,a})`, `b({m,f})` across all loaded modules (functions and callbacks, including a hand-built module with a documented `@macrocallback`) produced no `FunctionClauseError`/`CaseClauseError`; macrocallback arity translation (`arity - 1`) and doc lookup resolve correctly.

## Risks / caveats

- Two latent partial functions (`type_doc/5` and `rewrite_source/1`, detailed under *Ruled out*) rely on invariants supplied by the Elixir compiler (public types always have doc entries; stdlib sources always sit under `lib/`/`src/`). They are not reachable via any real module today, but a hand-crafted or non-Elixir BEAM that violates those invariants would turn them into a MatchError — and for `source_location/1` that raise would break its documented `{:ok, _} | {:error, _}` contract. Adding an `else`/fallback clause would make them total at negligible cost.
- I could not fetch the upstream source (WebFetch is not permitted in this session), so the one reported finding is judged purely on the internal inconsistency between `t/1` and `t/2`,`t/3`; if `t/1`'s inclusion of private types is intentional, downgrade it to non-issue.
- Coverage: dynamic sweeps used `:code.all_available()` (all discoverable modules) for `t`/`h`/`b`/`source_location`; the per-function `h({m,f,a})`/`b({m,f})` sweep was capped at the first 400 modules for time, so a defect specific to a function in a later-sorted module could in principle be missed, though the earlier all-module `h(m)`/`b(m)` and all-type `t` sweeps were unbounded.

