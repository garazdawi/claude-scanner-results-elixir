# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/kernel/special_forms.ex
<!-- sha256: db58059f0c8c36cea4b28ae9f56c5412f7d6b02379563fa93b771608f240cbdd -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T11:29:52Z_

Verification confirms the module's behavior. The key findings from running it:

- `Kernel.SpecialForms.for([1,2,3])` and `Kernel.SpecialForms.super(1)` both raise a `RuntimeError` with the **static** message — a fully controlled error.
- Critically, injecting `IO.puts("SIDE-EFFECT-RAN")` into the argument position did **not** print anything — proving the `_ = unquote(args)` idiom does not execute the argument expressions; it binds their AST as inert data and the macro raises at the caller's compile time before any argument code could run.

I've read all 2403 lines. The file is a moduledoc + a single private helper `error!/1` + a long list of one-line `defmacro` stubs, each delegating to `error!`. There are no other constructs.

---

# Security review — `lib/elixir/lib/kernel/special_forms.ex`

## Summary
`Kernel.SpecialForms` is a documentation-and-stub module whose every macro raises an identical, static-message `RuntimeError`; it contains no sink that processes external input, so there are no security findings.

## Security

`No findings`

The module consists entirely of `@moduledoc`/`@doc` text plus one private helper and a set of macro stubs. The only executable construct is:

```elixir
defmacrop error!(args) do
  quote do
    _ = unquote(args)
    message = "Elixir's special forms are expanded by the compiler and must not be invoked directly"
    :erlang.error(RuntimeError.exception(message))
  end
end
```

Every public macro (`{}`, `%{}`, `%`, `<<>>`, `.`, `alias`, `require`, `import`, `__ENV__`, `quote`, `unquote`, `for`, `with`, `case`, `try`, `receive`, `super`, `&`, `^`, …) has the body `error!([...])`. None of the Phase‑1 sink classes are present: there is no `Code.eval*`/`Module.create`, no `String.to_atom`/`Module.concat`, no `binary_to_term`, no `System.cmd`/`os.cmd`, no `File`/`Path` operation, no `Regex.compile`, no `:crypto`, no ETS/persistent_term/process-dictionary write, no network call, and no Unicode/identifier handling (those live in the tokenizer, not here). The one value that crosses into the macro — the caller's argument AST — is never used as a sink; it is discarded into `_` and the macro raises.

### Ruled out

- **Code execution / unquote hygiene via `_ = unquote(args)`** — One might suspect that inserting caller-supplied AST via `unquote(args)` executes it. It does not: `args` is a *list literal* built by this file (`[struct, map]`, `[expr]`, `[]`), and `_ = unquote(args)` only forces the parameters to be "used" so the compiler doesn't warn; the macro then raises at expansion time. **Verified** — `Code.eval_string("... Kernel.SpecialForms.for(IO.puts(\"SIDE-EFFECT-RAN\"))")` raised `RuntimeError` and never printed `SIDE-EFFECT-RAN`, i.e. the argument expression is never evaluated. No caller-controlled code is inserted or run.
- **Template / log injection in the raised message** — The `message` is a compile-time constant binary with no interpolation of arguments. **Verified** — the message is byte-for-byte identical (`"Elixir's special forms are expanded by the compiler and must not be invoked directly"`) regardless of the arguments passed. No injection surface.
- **Dead / unreachable defensive code** — The stubs are intercepted by the compiler for normal (unqualified) special-form syntax and are reached only via qualified remote-macro calls (e.g. `Kernel.SpecialForms.for(...)`), where raising is the intended contract. **Verified** — qualified calls to `for/1` and `super/1` raise the documented error. These are not defensive gates guarding a downstream sink (there is nothing behind them), so an "unreachable defence" has no security consequence here.
- **Atom exhaustion / dispatch** — No conversion of external bytes to atoms and no `apply`/dynamic dispatch anywhere in the file (whole file read).
- **Documentation example code** (`File.read`, `File.cp!`, `Logger.error("… #{path} …")`, `Code`-style snippets) — These appear only inside `@doc` strings as illustrations; the `\#{path}` in the `with` example is escaped so it is not even a doctest. Documentation is not executed as part of this module (whole file read; every code reference is inside a doc string).

## Risks / caveats
- **Scope of the "no sinks" claim:** I read the file in full (2403 lines). The claim quantifies over the entire file, whose only executable code is the `error!/1` helper and the delegating stubs — there is no other function body to hide a sink in.
- **Not the identifier/Unicode surface:** the folder threat model and Unicode/bidi/NFC obligations target the Erlang tokenizer under `lib/elixir/src`; this Elixir-level `SpecialForms` module performs none of that handling, so those checklist items are not applicable here (stated rather than force-matched).
- **Behavioral basis:** the reachability and no-side-effect claims were exercised against `elixir 1.21.0-dev` built from the source under review; I did not attempt to reach every one of the ~40 stubs individually, but they are structurally identical (each `do: error!([...])`), so the two exercised representatives (`for/1`, `super/1`) plus the side-effect probe are dispositive for the shared code path.

