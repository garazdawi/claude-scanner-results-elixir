# /Users/lukas/code/claude-scanner/elixir/lib/elixir/lib/record/extractor.ex
<!-- sha256: 2f7cf266c1f7f618524a3dfff34d22ff7970ce77fd3ebd2b9b857480b6ffad82 -->
<!-- tree: 3389f8405 2026-08-25 -->

_Analysed on 2026-08-28T12:27:52Z_

I have everything I need. All behaviors are verified by running against the source-built toolchain.

---

# Security Review — `lib/record/extractor.ex`

## Summary
`Record.Extractor` backs the public `Record.extract/2` / `Record.extract_all/1` API; extracting records from an Erlang file that is not fully trusted **executes attacker-supplied code** (record default-value expressions are evaluated via `:erl_eval.expr`), and an untrusted `:from_lib` value permanently leaks atoms.

## Security

#### `CSCAN-extractor-record-default-eval-rce`
- **Finding ID:** `CSCAN-extractor-record-default-eval-rce`
- **Severity:** High
- **Confidence:** high
- **Location:** `lib/record/extractor.ex:118` (`eval_record/2`), fed by `parse_record/2` (l.95–98) and `parse_field/1` (l.108–110); reached from `Record.extract/2` and `Record.extract_all/1`.
- **Class:** Compile-time execution / Code execution.
- **Issue:** `parse_record` copies each record field's *default-value expression* — arbitrary Erlang AST taken verbatim from the parsed file (`{:record_field, _, key, value}` → the raw `value` AST) — into a synthetic function, and `eval_record` runs it through `:erl_expand_records.module/2` and then `:erl_eval.expr(record_ast, [])`. `:erl_eval.expr` is called with **no local/non-local handler and no restrictions**, so any function call in a default value (e.g. `os:cmd/1`, `file:*`, `erlang:halt/0`) is executed with full VM privileges. There is no sandbox and no restriction to constant/guard-safe expressions.
- **Impact / threat:** `Record.extract/…` is normally invoked *at compile time* to read record definitions out of Erlang headers. If the header being extracted — or anything it pulls in via `-include`/`-include_lib` (epp expands these) — is attacker-influenced, the attacker gets **code execution on the build machine**. The realistic boundary is supply chain: `Record.extract(from_lib: "some_erlang_dep/include/x.hrl")` runs that dependency's code during *your* compile even though a header looks like a pure declaration and you never call the dependency. The trust boundary crossed is "a file the project merely reads" → code sink; the build path has wide reach and weak expectations.
- **Evidence:** `verified`.
```
$ cat /tmp/rectest/evil.hrl
-record(evil, {pwned = os:cmd("echo CODE_EXECUTION_PROOF_$$ > /tmp/rectest/PWNED.txt"), normal = 1}).
$ elixir -e 'IO.inspect(Record.extract(:evil, from: "/tmp/rectest/evil.hrl"))'
[pwned: [], normal: 1]
$ cat /tmp/rectest/PWNED.txt
CODE_EXECUTION_PROOF_246          # the shell command in the record default ran
```
- **Suggested fix:** Do not evaluate defaults with an unrestricted evaluator. Evaluate `:erl_eval.expr/4` with a `non_local`/local function handler that rejects (or allow-lists to pure) function calls, or first reject any default that is not a constant/guard-safe expression (e.g. via `:erl_lint`), returning the AST unevaluated for the rest. At minimum, document that `Record.extract` executes code from the target file and must only be pointed at trusted Erlang sources.

#### `CSCAN-extractor-from-lib-atom-exhaustion`
- **Finding ID:** `CSCAN-extractor-from-lib-atom-exhaustion`
- **Severity:** Low
- **Confidence:** high
- **Location:** `lib/record/extractor.ex:43` (`from_lib_file/1`), reached from `Record.extract/2` / `Record.extract_all/1` with `:from_lib`.
- **Class:** Atom creation (CWE-770).
- **Issue:** `from_lib_file` splits the `:from_lib` string and calls `:code.lib_dir(List.to_unsafe_atom(app))`. `List.to_unsafe_atom/1` is `:erlang.list_to_atom/1`; it creates a **new, never-collected atom** for every distinct first path segment. The atom is created *before* `:code.lib_dir` runs, so even a lookup that fails (`{:error, _}` → `raise`) still leaks the atom. The `@spec` types `from_lib` as `binary()`; passing a dynamically-computed binary does not violate the spec (the "literals at compile time" note is guidance, not a type contract).
- **Impact / threat:** If any caller forwards untrusted data into `:from_lib` (a tool extracting user-specified libraries at run time), each unique app-name segment permanently consumes an atom-table slot; ~1M distinct values exhaust the default table and crash the VM. Lower severity because the common use is compile-time literals and `:code.lib_dir` genuinely needs an atom.
- **Evidence:** `verified`.
```
$ elixir -e '... 200 failed from_lib lookups with unique app names ...'
atoms before=13771 after=14291 delta=520
zzznotanapp42 now exists? true     # atom persisted despite the lookup raising
```
- **Suggested fix:** Validate the app segment against known applications before atomizing (e.g. look it up among `:code.get_path`/`Application` names), or use `List.to_existing_atom/1` with a graceful "lib not found" fallback so unknown names never mint atoms.

### Ruled out
- **Arbitrary file path / traversal via `:from`/`:from_lib` → `:epp.parse_file` (l.83).** Verified that absolute paths and `../` are honored with no confinement (`from: "/tmp/rectest/fakepasswd"` and `from: "../../../../etc/hostname"` were opened/parsed). Not reported as a read primitive: file *contents are never returned* to the caller (only extracted records or a "no record found" error), the error text does not echo file content (`:epp.parse_file` returns `{:ok, Forms}` even for non-Erlang files, so the content-bearing `raise` branch rarely fires), and reading a developer-named Erlang file is the feature's stated purpose. On this same input the reported RCE/atom findings dominate.
- **`opts` forwarded to `:epp.parse_file` (`:includes`, `:macros`, remaining keys).** These are caller-supplied compiler options, not data from the external file; no external-trust boundary is crossed within this file.
- **`from_file/1` `:code.where_is_file` (l.33).** Pure code-path lookup with no atomization or injection sink; falls through to the path handling already covered above.
- **`List.to_unsafe_atom` as an injected defect.** Confirmed the file matches its committed upstream form; `to_unsafe_atom` is the current (1.21) rename of the now-deprecated `List.to_atom` and is semantically identical — the atom concern is pre-existing design, not a modification.

## Risks / caveats
- **The RCE is inherent upstream behavior, not a modification to this file.** The file matches its committed tree (only `List.to_atom` → `List.to_unsafe_atom`, a semantics-preserving deprecation rename). Evaluating defaults is *partly* necessary — you cannot obtain the value of `f = 1 + 2` without evaluating — so a fix must distinguish safe constant/guard expressions from arbitrary calls rather than dropping evaluation wholesale; that carries a compatibility cost. I rated it High on impact (build-machine code execution reachable from a dependency-supplied file) while noting the precondition below.
- **Precondition realism.** Both primary findings require a caller to point extraction at attacker-influenced input (`from`/`from_lib` string, or the target/`-include`d file's contents). The documented, dominant use is compile-time literals against trusted OTP/dependency headers; that use is safe. The findings matter specifically for supply-chain (extracting from third-party Erlang libraries) and for any tooling that feeds run-time/user-controlled values in. A caller who only ever passes vetted literal paths to vetted files is not exposed.
- **Not independently re-derived:** that `:erl_expand_records.module/2` performs no evaluation itself (I treated only `:erl_eval.expr` as the eval sink) and the exact atom-table default limit; both are standard OTP behavior and not load-bearing for the findings, which are demonstrated by direct execution against the source-built toolchain.

