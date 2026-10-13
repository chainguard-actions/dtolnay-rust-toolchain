<!-- markdownlint-disable -->

# Hardening Report: dtolnay--rust-toolchain/v1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **dtolnay--rust-toolchain/v1** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are directly interpolated inside `run:` shell command strings in action.yml (sub-rule a). This applies even to `runner.*` context values, which flow through YAML template substitution before the shell sees them.

1. Line 41 ("Parse toolchain version" step): `if [[ ${{runner.os}} == macOS ]]; then`
2. Line 60 ("Set $CARGO_HOME" step): `run: echo CARGO_HOME=${CARGO_HOME:-"${{runner.os == 'Windows' && '$USERPROFILE\.cargo' || '$HOME/.cargo'}}"}  >> $GITHUB_ENV`
3. Line 75 ("Install rustup if needed on windows" step): `curl ... https://win.rustup.rs/${{runner.arch == 'ARM64' && 'aarch64' || 'x86_64'}} --output '${{runner.temp}}\rustup-init.exe'`
4. Line 76 ("Install rustup if needed on windows" step): `'${{runner.temp}}\rustup-init.exe' --default-toolchain none --no-modify-path -y`
5. Lines 145/147/150 ("Enable Cargo sparse registry" step): `"${{runner.temp}}"` used in file path tests and touch commands

Locations:

- `action.yml:41`
- `action.yml:60`
- `action.yml:75`
- `action.yml:76`
- `action.yml:145`

### github-env-injection (severity: high)

The "Set $CARGO_HOME" step (line 60) writes a value derived from the `${{runner.os == 'Windows' && ...}}` expression directly to `$GITHUB_ENV` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). The expression is interpolated inline in the `run:` command and its result is written unsanitized to the special environment file, allowing newline injection into the environment.

Locations:

- `action.yml:60`

### unsafe-shell (severity: high)

The "Install rustup if needed" step pipes remote content directly to a shell interpreter without first downloading to a file: `curl --proto '=https' --tlsv1.2 ... https://sh.rustup.rs | sh -s -- --default-toolchain none -y`. Even though TLS is enforced, this pattern executes whatever the remote server returns without any opportunity to inspect or verify the script content.

Locations:

- `action.yml:66`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all ${{ }} expressions from run: shell strings into env: blocks and referenced them as plain environment variables:
   - ${{runner.os}} → RUNNER_OS env var (parse step)
   - ${{runner.os == 'Windows' && ...}} → RUNNER_OS_CARGO_HOME env var (Set $CARGO_HOME step)
   - ${{runner.arch == 'ARM64' && ...}} → RUSTUP_ARCH env var (Windows rustup install step)
   - ${{runner.temp}} → RUNNER_TEMP env var (Windows rustup install step and Enable Cargo sparse registry step)

2. github-env-injection: Rewrote the Set $CARGO_HOME step to sanitize the value with `printf '%s' ... | tr -d '\n\r'` before writing to $GITHUB_ENV.

3. unsafe-shell: Rewrote the Install rustup if needed step to download the script to a temp file (mktemp) first, then execute it with `sh "$RUSTUP_INIT" --default-toolchain none -y` (dropped the `--` shell option terminator since it's no longer piped to sh -s). The temp file is cleaned up after execution.

### Iteration 2

**Fixes applied:** github-env-injection, script-injection

**Notes:**

1. github-env-injection: Added 'toolchain=$(printf \'%s\' "$toolchain" | tr -d \'\\n\\r\')' at the very start of the 'Parse toolchain version' step's run script. This sanitizes the user-controlled $toolchain variable by stripping newlines/carriage returns before it is used in any branch that writes to $GITHUB_OUTPUT, preventing injection of arbitrary key=value pairs. The heredoc (else) branch is also protected since the sanitized value can no longer contain newlines that would escape the delimiter.
2. script-injection: Replaced the unquoted 'for t in ${targets//,/ }' and 'for c in ${components//,/ }' loops with proper xargs-based array tokenization. Each list is guarded with '[ -n "$VAR" ]' to prevent xargs from emitting an empty token on empty input, then tokenized with 'printf \'%s\' "${VAR//,/ }" | xargs printf \'%s\\0\'' and read into bash arrays via 'while IFS= read -r -d \'\'\' token'. The 'set -f' was removed as glob expansion is no longer a concern with this approach.

