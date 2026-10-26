# Cluster cross-file findings

#### `CSCAN-cluster-logger-reports-value-validation-divergence`

- **Finding ID** — `CSCAN-cluster-logger-reports-value-validation-divergence`
- **Severity** — Low
- **Confidence** — high
- **Files involved** — `elixir` (lines 133-144), `elixir.bat` (lines 126-127)
- **Finding shape** — Windows/Unix parity divergence (WINDOWS_PARITY, sub-claim 9)
- **Evidence** —
  - `elixir:133-138` gates the emission on the value being a literal boolean:
    ```
    --logger-otp-reports)
        S=2
        if [ "$2" = 'true' ] || [ "$2" = 'false' ]; then
          ERL="$ERL -logger handle_otp_reports $2"
        fi
    ```
    So `elixir --logger-otp-reports maybe` on Unix consumes both tokens and appends **nothing** to the erl command line.
  - `elixir.bat:126` appends unconditionally, with no `true`/`false` check:
    ```
    if ""==!par:--logger-otp-reports=!  (set "parsErlang=!parsErlang! -logger handle_otp_reports %1" && shift && goto startloop)
    ```
    So the same invocation on Windows passes `-logger handle_otp_reports maybe` straight to `erl.exe`, where the kernel logger config expects a boolean and will fault or mis-configure. `--logger-sasl-reports` (`elixir:139-143` vs `elixir.bat:127`) diverges identically.
- **What the per-file pass missed** — Each script is internally self-consistent; only comparing the two side-by-side reveals that the `sh` variant silently drops a non-boolean value while the `.bat` variant forwards it to the VM.
- **Suggested fix** — In `elixir.bat`, guard the two logger arms with an `if /I "%~1"=="true" ... else if /I "%~1"=="false"` check mirroring `elixir:135`, appending the `-logger …` switch only for boolean values (and `shift`ing the value in all cases to keep arity aligned).

#### `CSCAN-cluster-erl-switches-leak-into-extra-on-windows`

- **Finding ID** — `CSCAN-cluster-erl-switches-leak-into-extra-on-windows`
- **Severity** — Low
- **Confidence** — medium
- **Files involved** — `elixir` (lines 149-208, 211-216, 229), `elixir.bat` (lines 99, 121-131, 143-145)
- **Finding shape** — Inherited-assumption / parity divergence: the two launchers hand `elixir`'s CLI (`elixir_cli`) a different `-extra` argv for the same input
- **Evidence** —
  - In `elixir`, the distribution/release/`--erl` switches are consumed by **shift** (`S=…`), never copied into the `-extra` tail: e.g. `--cookie` (`149-153`), `--name`/`--sname` (`154-158`), `--boot` (`169-173`), `--vm-args` (`164-168`), `--erl` (`145-148`). `elixir:207-208` (`I=$((I+S)); shift $S`) drops them from `$@`; the erl-native forms are prepended via `erl_set`/`$ERL` **before** `-extra` at `elixir:211-216` and `elixir:229`. Net effect: elixir's `-extra` program argv never contains `--cookie`, `--name`, `--boot`, `--erl`, etc.
  - In `elixir.bat`, the same switches are added to `parsErlang` (`elixir.bat:121-131`) **but the final exec uses `%*`** (`elixir.bat:143` and `145`: `… %parsErlang% %beforeExtra% -extra %*`). Because cmd.exe `%*` always expands to the *original* argv regardless of the intervening `shift`s (`elixir.bat:99` and the per-arm `shift`s), every one of those switches — `--cookie value`, `--name node`, `--boot file`, `--erl "-foo"`, … — is *also* re-passed after `-extra`, i.e. into elixir's program argv.
- **What the per-file pass missed** — The Unix reviewer sees `S=`-consumed switches vanish and the Windows reviewer sees `%*` as an "intentional" pass-through; only diffing the two flag tables shows that the erl-only switches reach `elixir_cli`'s `-extra` stream on Windows but not on Unix, so identical command lines yield different program-side argv across platforms.
- **Suggested fix** — Reconstruct the trailing program arguments explicitly on Windows (accumulate the non-consumed/`*)`-class tokens into a dedicated variable as they are parsed) and emit that variable after `-extra` instead of `%*`, so distribution/release/`--erl` switches are not re-presented to `elixir_cli` — matching the `sh` script's behavior.
