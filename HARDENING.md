<!-- markdownlint-disable -->

# Hardening Report: dtolnay--rust-toolchain/v1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **dtolnay--rust-toolchain/v1** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Multiple `${{ }}` expressions are directly interpolated inside `run:` shell command strings. Any `${{ ... }}` in a `run:` block is a script-injection risk because the value is substituted by the YAML template engine before the shell ever sees it, allowing injection of shell metacharacters.

1. Step 'Parse toolchain version' (line 35): `if [[ ${{runner.os}} == macOS ]];`
2. Step 'Set $CARGO_HOME' (line 55): `echo CARGO_HOME=${CARGO_HOME:-"${{runner.os == 'Windows' && '$USERPROFILE\.cargo' || '$HOME/.cargo'}}"} >> $GITHUB_ENV`
3. Step 'Install rustup if needed on windows' (line 70): `curl ... https://win.rustup.rs/${{runner.arch == 'ARM64' && 'aarch64' || 'x86_64'}} --output '${{runner.temp}}\rustup-init.exe'` and `'${{runner.temp}}\rustup-init.exe'`
4. Step 'Enable Cargo sparse registry' (lines 107, 109, 112): `"${{runner.temp}}"` used in condition and `touch` commands inside the run block.

Locations:

- `action.yml:35`
- `action.yml:55`
- `action.yml:70`
- `action.yml:71`
- `action.yml:107`
- `action.yml:109`
- `action.yml:112`

### github-env-injection (severity: high)

The 'Set $CARGO_HOME' step writes a value derived from a `${{ }}` expression directly to `$GITHUB_ENV` without sanitization. The expression `${{runner.os == 'Windows' && '$USERPROFILE\.cargo' || '$HOME/.cargo'}}` is interpolated by the YAML template engine into the shell command string and then written unsanitized to `$GITHUB_ENV`. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write.

Offending line: `run: echo CARGO_HOME=${CARGO_HOME:-"${{runner.os == 'Windows' && '$USERPROFILE\.cargo' || '$HOME/.cargo'}}"} >> $GITHUB_ENV`

Locations:

- `action.yml:55`

### unsafe-shell (severity: high)

The 'Install rustup if needed' step pipes remote content directly to a shell interpreter without first downloading and verifying the script. Pattern matched: `curl --proto '=https' --tlsv1.2 ... https://sh.rustup.rs | sh -s -- --default-toolchain none -y`. If the remote server or network path is compromised, arbitrary code executes on the runner.

Locations:

- `action.yml:62`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. **script-injection**: Replaced all `${{runner.os}}` (line 35), `${{runner.arch}}` and `${{runner.temp}}` (lines 70-71), and `${{runner.temp}}` (lines 107, 109, 112) in `run:` blocks with their equivalent pre-set environment variables `$RUNNER_OS`, `$RUNNER_ARCH`, and `$RUNNER_TEMP`. These are standard GitHub Actions runner environment variables that are already safe shell variables.

2. **github-env-injection**: Rewrote the `Set $CARGO_HOME` step (line 55) to use shell logic (`if [[ $RUNNER_OS == Windows ]]`) instead of a `${{ }}` expression, and sanitized the value with `printf '%s' ... | tr -d '\n\r'` before writing to `$GITHUB_ENV`.

3. **unsafe-shell**: Fixed the `curl ... | sh -s -- --default-toolchain none -y` pattern (line 62) by downloading the rustup installer to `$RUNNER_TEMP/rustup-init.sh` first, then executing it with `sh "$RUNNER_TEMP/rustup-init.sh" --default-toolchain none -y` (dropping the `--` shell option terminator as required by the fix guidelines).

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

Fixed two high-severity findings in hardened/action/action.yml:

1. github-env-injection (line 53): In the 'Parse toolchain version' step's else branch, added sanitization of the $toolchain value before writing it to GITHUB_OUTPUT using the heredoc form. Added `safe_toolchain=$(printf '%s' "$toolchain" | tr -d '\n\r')` and replaced `$toolchain` with `$safe_toolchain` in the printf statement. This prevents newline injection into GITHUB_OUTPUT.

2. script-injection (lines 99-100): In the 'rustup toolchain install' step, replaced the unquoted `${targets//,/ }` and `${components//,/ }` expansions (which allowed shell metacharacters to influence tokenization) with proper xargs-based tokenization. Each list input is now processed with a guard (`if [ -n "$VAR" ]`), xargs -r for safe tokenization, and a NUL-delimited read loop to safely populate the flags array. The comma-to-space substitution is preserved inside the printf argument before piping to xargs.

