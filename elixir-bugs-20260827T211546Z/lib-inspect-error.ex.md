# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/inspect/error.ex
<!-- sha256: ae1babeaa7736c0d86e689fc5592430b011d6b698579672d9e980fd34173956c -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-27T22:15:40Z_

## Summary

The empty-stacktrace branch of the `Inspect` implementation double-indents the first line of the error message, producing visibly misaligned output; no other reachable defects were found.

## Bugfixes

#### `CSCAN-error-empty-stacktrace-double-indent`
- **Finding ID** — `CSCAN-error-empty-stacktrace-double-indent`
- **Severity** — Low
- **Confidence** — high
- **Location** — `lib/inspect/error.ex:86`, `format_output(message, [])` clause of `defimpl Inspect, for: Inspect.Error`
- **Issue** — The interpolation is written `  #{Inspect.Error.pad(message, 2)}` with two leading template spaces, while `pad/2` already prepends two spaces to *every* line of `message`. The first message line therefore receives 4 spaces of indentation and all subsequent lines receive only 2. The non-empty-stacktrace clause (line 74) correctly writes `#{Inspect.Error.pad(message, 2)}` with no leading spaces, so the two clauses render the same message inconsistently.
- **Class** — Off-by-one / wrong arithmetic (indentation width)
- **Impact** — When an `%Inspect.Error{}` with an empty `stacktrace` is inspected, its (always multi-line) message is rendered ragged: the `got … with message:` header sits at column 4 while `while inspecting:` sits at column 2. Cosmetic only — wrong indentation in a diagnostic, no crash or data loss.
- **Evidence** — `verified`. Ran (with `ELIXIR_ERL_OPTIONS=+fnu`):
  ```
  err = %Inspect.Error{exception_module: RuntimeError,
    exception_message: "kaboom\nsecond line", stacktrace: [],
    inspected_struct: "%Boom{a: 1}"}
  IO.puts(inspect(err))
  ```
  Output (empty-stacktrace branch):
  ```
  #Inspect.Error<
      got RuntimeError with message:      <- 4 spaces

        """
        kaboom
        ...
    while inspecting:                     <- 2 spaces

        %Boom{a: 1}
  >
  ```
  Compare the non-empty branch (via `inspect(struct(Boom, a: 1), safe: true)`), where `got RuntimeError with message:` is indented 2 spaces uniformly with the rest of the body.
- **Suggested fix** — Drop the two leading spaces on line 86 so it reads `#{Inspect.Error.pad(message, 2)}`, matching line 74.

## Risks / caveats

- The empty-stacktrace clause (`format_output/2` with `[]`) is not reachable through the normal inspect-failure path in `Inspect.Algebra.to_doc_with_opts/2`, because that path always supplies `__STACKTRACE__`, which is non-empty. It is reachable only by inspecting a manually constructed `%Inspect.Error{stacktrace: []}`. That construction uses the public exception struct via the public `inspect/2`, but the clause's practical exposure is low; hence Low severity.
- I could not exercise the file through a from-source rebuild of a modified copy; testing used the installed `/opt/elixir` build, which self-reports as the same tree (`elixir 1.21.0-dev`, stacktraces cite `lib/inspect/algebra.ex` at the reviewed line numbers), so observed behavior reflects this source.

### Ruled out
- **`pad/2` crashing on non-binary `inspected_struct`/`exception_message`** — the only documented producer, `Inspect.Algebra.to_doc_with_opts/2` (`lib/inspect/algebra.ex:423-433`), always passes an `IO.iodata_to_binary/1` result for `inspected_struct` and `Exception.message/1` (always a binary) for `exception_message`, so `pad`'s `is_binary` guard is always satisfied on the public path. Basis: `verified` (traced the sole caller; ran the safe-inspect path successfully).
- **`pad/2` empty-line handling dropping/adding stray newlines** — tested `"a\n\nb"`, `"a\n"`, and `""`; blank lines are emitted without padding and the trailing newline is trimmed as intended, output correct. Basis: `verified`.
- **`message/1` partial map match missing `stacktrace`/`exception_module`** — partial map patterns match structs with additional keys fine; `Exception.message/1` supplies a full `%Inspect.Error{}`. Basis: `verified` (ran `Exception.message(err)`).
- **`exception/1` `Keyword.fetch!` raising on missing keys** — raising `KeyError` on an argument list lacking `:exception`/`:stacktrace`/`:inspected_struct` is the documented contract (arguments is a keyword list); not a defect. Basis: `asserted` (contract reading).

