<!-- markdownlint-disable -->

# Hardening Report: dtolnay--rust-toolchain/v1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **dtolnay--rust-toolchain/v1** was hardened automatically. 10 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Install rustup if needed' step downloads a script and pipes it straight into a shell: `curl ... https://sh.rustup.rs | sh -s -- --default-toolchain none -y`. Download the installer to a file first, then run it as a separate command.

Locations:

- `action.yml:66`

### script-injection (severity: high)

Sub-rule (a): the 'Parse toolchain version' step puts an expression directly in the run script: `if [[ ${{runner.os}} == macOS ]]; then`. Use "$RUNNER_OS" instead.

Locations:

- `action.yml:41`

### script-injection (severity: high)

Sub-rule (a): the 'Set $CARGO_HOME' step puts an expression directly in the run script: `echo CARGO_HOME=${CARGO_HOME:-"${{runner.os == 'Windows' && '$USERPROFILE\.cargo' || '$HOME/.cargo'}}"} >> $GITHUB_ENV`.

Locations:

- `action.yml:60`

### script-injection (severity: high)

Sub-rule (a): the 'Install rustup if needed on windows' step puts expressions directly in the run script: `https://win.rustup.rs/${{runner.arch == 'ARM64' && 'aarch64' || 'x86_64'}} --output '${{runner.temp}}\rustup-init.exe'` and `'${{runner.temp}}\rustup-init.exe' ...`. Use $RUNNER_ARCH and "$RUNNER_TEMP" instead.

Locations:

- `action.yml:75`
- `action.yml:76`

### script-injection (severity: high)

Sub-rule (a): the 'Enable Cargo sparse registry' step puts an expression directly in the run script several times: `-f "${{runner.temp}}"/.implicit_cargo_registries_crates_io_protocol` and `touch "${{runner.temp}}"/...`. Use "$RUNNER_TEMP" instead.

Locations:

- `action.yml:145`
- `action.yml:147`

### script-injection (severity: high)

Sub-rule (b): the 'rustup toolchain install' step expands variables that come from inputs without double quotes: `for t in ${targets//,/ }; do`, `for c in ${components//,/ }; do`, and `[[ $toolchain == nightly && -n $components ]]`. The split on commas is deliberate, and `set -f` turns off glob expansion. A fix that keeps this behaviour is to read the values into arrays with `IFS=', ' read -ra` and then expand the arrays with quotes.

Locations:

- `action.yml:86`
- `action.yml:87`
- `action.yml:88`

### github-env-injection (severity: high)

The 'Parse toolchain version' step writes the value of inputs.toolchain (passed in through env `toolchain`) to $GITHUB_OUTPUT with `printf 'toolchain<<%s\n%s\n%s\n' "$delimiter" "$toolchain" "$delimiter" >> $GITHUB_OUTPUT`. The step does pick a heredoc delimiter that does not appear in the value, but it does not apply the required `printf '%s' "$toolchain" | tr -d '\n\r'` sanitization before the write.

Locations:

- `action.yml:53`

### github-env-injection (severity: high)

The 'Set $CARGO_HOME' step copies CARGO_HOME, which the calling workflow can set, into $GITHUB_ENV without sanitizing it: `echo CARGO_HOME=${CARGO_HOME:-...} >> $GITHUB_ENV`.

Locations:

- `action.yml:60`

### github-env-injection (severity: high)

The 'Install rustup if needed' step writes the inherited CARGO_HOME to $GITHUB_PATH without sanitizing it: `echo "$CARGO_HOME/bin" >> $GITHUB_PATH`.

Locations:

- `action.yml:67`

### github-env-injection (severity: high)

The 'Install rustup if needed on windows' step writes the inherited CARGO_HOME to $GITHUB_PATH without sanitizing it: `echo "$CARGO_HOME\bin" >> $GITHUB_PATH`.

Locations:

- `action.yml:77`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

I fixed all the findings in hardened/action/action.yml. I haven't run the action or its tests, so none of these changes are tested yet.

- **Rustup installer (unsafe-shell):** the script from sh.rustup.rs is now saved to $RUNNER_TEMP/rustup-init.sh, run with `sh "$rustup_init" --default-toolchain none -y`, then deleted. I dropped the `-s --` because those options only made sense when the shell read the script from stdin.
- **Script injection, runner values:** every `${{runner.*}}` in run scripts is replaced with the shell variables $RUNNER_OS, $RUNNER_ARCH and $RUNNER_TEMP. The Windows/Linux CARGO_HOME default and the Windows ARM64/x86_64 download URL are now picked with bash if-statements instead of expressions.
- **Script injection, targets and components:** these are now split into arrays with `read -r -d '' -a`, skipped entirely when empty, and expanded in quotes. This still splits on commas and any whitespace, as upstream did. I used this instead of the suggested `IFS=', ' read -ra` because that form reads only the first line.
- **Outputs and PATH (github-env-injection):** the toolchain output, CARGO_HOME (written to $GITHUB_ENV) and both $GITHUB_PATH writes now have newlines stripped with `tr -d '\n\r'` before writing.

One behaviour change: the toolchain output used to be written in the multi-line (heredoc) form and is now a plain `toolchain=<value>` line. A toolchain name is a single line, so this should not matter in practice, but a value containing newlines would now have them merged.

### Iteration 2

**Fixes applied:** script-injection, invalid-yaml

**Notes:**

Rewrote action.yml. The previous pass had left it corrupted: lines were merged, the 'Set $CARGO_HOME' run: had the wrong indentation, the curl lines in the two rustup-install steps were broken, both steps had lost their `if: runner.os` conditions, and the $RUNNER_TEMP paths in the sparse-registry step were mangled. I put back upstream's logic and kept the earlier hardening: the downloaded installer runs from a file with the '-s --' dropped, and values are sanitized before they are written to GITHUB_ENV, GITHUB_PATH and GITHUB_OUTPUT. Parse step: every $toolchain test is now double-quoted, and the here-string is `<<< "$toolchain"`. Command substitutions are stored in variables first. Arithmetic now uses variable names (minus, i, secs, now) instead of unquoted expansions. Install step: targets and components are still split on commas and whitespace as upstream did, now with `read -r -d '' -a arr <<< "${var//,/ }" || true`. Each split runs only when the value is non-empty, and its array is expanded quoted. This replaces the `set -f` plus unquoted for-loop. The nightly/components test is quoted too. I also quoted $GITHUB_OUTPUT and $GITHUB_ENV everywhere. I did not run the action or any tests after the rewrite.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

In the parse step of action.yml, I added '|| true' to five captured commands: the sed for 'ago' (line 42), the macOS date for 'secs' (line 43), the Linux date for 'secs' (line 46) and both date captures for 'now' (lines 51 and 55). Upstream ran these commands inside echo "$(...)", so a failure did not stop the step. With '|| true' the hardened version behaves the same way, and the output is still captured. I changed nothing else and did not re-run the publish checks.

### Iteration 2

**Fixes applied:** publish-gate

**Notes:**

In action.yml, I added `|| true` to the macOS `ago="$(sed ...)"` and `secs="$(date -v- ...)"` assignments (lines 42–43). Upstream ran these inside echo's arguments, so a failure didn't stop the step, and this keeps that behaviour. Nothing else changed. I haven't run the publish checks again on the result.

