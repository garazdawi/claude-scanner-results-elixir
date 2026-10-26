# /Users/lukas/code/claude-scanner/elixir/lib/elixir/scripts/docs_config.exs
<!-- sha256: dd0d033db30da9b96100e9929cb584345e4a70518af62719fb1cceea03c37c0b -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:58:06Z_

I have a complete, verified picture. Let me compose the report.

## Analysis complete

I traced every sink in the file, identified the trust boundaries, and reproduced the relevant behaviors against Elixir built from the source under review. The findings below reflect that this is a **first-party build script** (`elixir docs_config.exs <app>`), invoked by Elixir's own `Makefile`/docs target with a hardcoded app name and operating on the repository's own git tags — not a library or tool that runs on untrusted repositories.

---

# Security review — `lib/elixir/scripts/docs_config.exs`

## Summary
A small first-party build script that generates ExDoc's version-chooser `docs_config.js`; all of its sinks are driven by build-controlled inputs (the `app` argv and the repo's own git tags), and none are reachable from untrusted data through its documented invocation, so there are no actionable security findings.

## Security

`No findings`

### Ruled out

- **Command execution — `System.cmd("git", ["tag"])` (line 16).** Fixed argv `["tag"]` with no interpolated input and no shell; `System.cmd/3` executes the binary directly. Basis: `verified` — a git tag containing `$(touch INJECTED)` produced no file (and the output is consumed via `String.split`, never a shell). No injection surface.

- **Path traversal via `app` into `File.mkdir_p!`/`File.write!` (lines 63, 65).** The path component `app` is not sanitized, and an `app` of `"../../pwned"` does escape the `doc/` directory. Basis: `verified` — in a `/tmp` sandbox, `File.write!("doc/#{app}/docs_config.js", ...)` with `app="../../pwned"` wrote outside `doc/`. **Ruled out on reachability, not mechanism:** `app` is the first CLI argument, supplied by the trusted build system (Elixir's `Makefile` passes a literal app name from `eex elixir ex_unit iex logger mix`). No dependency, archive, git ref, or project-read file influences it; reaching this sink requires the trusted caller to pass a contract-violating argument, which is outside the threat model.

- **JS/JSON injection via `app` or version strings into the generated `.js` (lines 48, 66–68).** `JSON.encode_to_iodata!` does **not** escape `/`, `<`, `>`, or the raw line/paragraph separators U+2028/U+2029. Basis: `verified` — encoding `"</script><script>…"` and raw U+2028 passed them through unescaped. **Ruled out** because the values that reach the encoder cannot carry those bytes and/or are trusted: version strings are round-tripped through `Version.parse!`, which rejects `"`, `<`, and U+2028 (`verified`: all three raise `invalid version`), so a crafted git tag cannot inject; and `app` is a trusted build literal. The output is also a standalone `.js` file loaded via `<script src>`, so `</script>` is not an inline-HTML break-out here.

- **Git-tag → `Version.parse!` build crash (line 21).** `version = Version.parse!(rest)` is a match expression inside the comprehension, so a `v`-prefixed tag that is not a valid three-part version raises rather than being filtered out, aborting the whole script. Basis: `verified` — a tag set including `v2.0` raised `invalid version: "2.0"` from the exact comprehension. **Ruled out as a security finding:** impact is availability-only (the docs build fails), and for this first-party script the tags are Elixir's own release tags — no untrusted party supplies them in the documented workflow. Noted in caveats.

- **`Enum.fetch!(0)` / `[app] = System.argv()` crashes (lines 5, 36).** Empty `versions` or a wrong argv arity raises. Robustness only, no trust boundary crossed; not security-relevant.

- **`app` shadowing in the `search_nodes` comprehension (line 59).** `for app <- ~w(...)` does not leak into the outer scope, so the `app` used for the file path remains the argv value. Basis: `verified` — the outer `app` was unchanged after the comprehension. No behavioral or security effect.

## Risks / caveats
- **Scope assumption:** I treated the script's "documented public API" as its CLI invocation `elixir docs_config.exs <app>` by Elixir's own build, with `app` and git tags as build-trusted. This is consistent with the file being hardcoded to Elixir's app list and `hexdocs.pm`. If this script were ever repurposed to run against an **untrusted repository** (attacker-controlled git tags) or with an **untrusted `app` argument**, two issues become live: (1) a crafted `v`-prefixed non-version tag crashes the build (verified), and (2) an `app` containing `../` traverses out of `doc/` on write (verified). Both are cheap to harden — filter tags with `Version.parse/1` instead of `Version.parse!/1`, and validate `app` against the known app allow-list (or `Path.safe_relative/1`) before using it in a path — but neither is reachable from untrusted input in the documented use.
- I did not have the invoking `Makefile` in this partial checkout; the invocation model is inferred from the sibling `elixir_docs.exs` and the hardcoded, Elixir-specific constants in this file. That inference is the sole basis for treating `app` as trusted; it is well-supported but not read from the build definition directly.
- `JSON.encode_to_iodata!` not escaping `<`/`/`/U+2028 is a property of the encoder, not this file; it is only benign here because the reachable inputs are constrained. Any future change that lets a less-constrained string into `version_nodes`/`search_nodes` would reopen the JS-context question.

