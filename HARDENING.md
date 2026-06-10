<!-- markdownlint-disable -->

# Hardening Report: dtolnay--rust-toolchain/v1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **dtolnay--rust-toolchain/v1** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings throughout action.yml, violating rule (a). This includes: line 39 `if [[ ${{runner.os}} == macOS ]]`; line 57 `echo "downgrade=${{steps.parse.outputs.toolchain == 'nightly' && inputs.components && ...}}" >> $GITHUB_OUTPUT`; line 63 `echo CARGO_HOME=${CARGO_HOME:-"${{runner.os == 'Windows' ...}}"} >> $GITHUB_ENV`; line 74 `curl .../win.rustup.rs/${{runner.arch == 'ARM64' && 'aarch64' || 'x86_64'}}`; line 75 `'${{runner.temp}}\rustup-init.exe'`; line 80 `rustup toolchain install ${{steps.parse.outputs.toolchain}}${{steps.flags.outputs.targets}}${{steps.flags.outputs.components}} --profile minimal${{steps.flags.outputs.downgrade}}`; line 83 `rustup default ${{steps.parse.outputs.toolchain}}`; lines 87-88 `rustc +${{steps.parse.outputs.toolchain}}`; lines 99-100 `${{runner.temp}}` and `${{steps.parse.outputs.toolchain}}`; lines 110 and 114 `${{steps.parse.outputs.toolchain}}`. All of these allow YAML template substitution to inject arbitrary shell metacharacters before the shell ever sees the value.

Locations:

- `action.yml:39`
- `action.yml:57`
- `action.yml:63`
- `action.yml:74`
- `action.yml:75`
- `action.yml:80`
- `action.yml:83`
- `action.yml:87`
- `action.yml:88`
- `action.yml:99`
- `action.yml:100`
- `action.yml:110`
- `action.yml:114`

### github-env-injection (severity: high)

Multiple steps write values derived from untrusted inputs to GITHUB_OUTPUT, GITHUB_ENV, or GITHUB_PATH without the required sanitization step (printf '%s' ... | tr -d '\n\r'). (a) The parse step writes $toolchain (from inputs.toolchain via env var) to $GITHUB_OUTPUT with bare echo statements - no newline stripping. (b) The flags step writes ${targets//,/ } and ${components//,/ } (from inputs.targets, inputs.target, inputs.components) to $GITHUB_OUTPUT without sanitization. (c) The flags step writes an expression involving inputs.components directly to $GITHUB_OUTPUT. (d) The CARGO_HOME step writes a runner.os expression result directly to $GITHUB_ENV without sanitization of the inherited $CARGO_HOME env var.

Locations:

- `action.yml:40`
- `action.yml:55`
- `action.yml:56`
- `action.yml:57`
- `action.yml:63`

### unsafe-shell (severity: high)

The install-rustup step pipes remote content directly to a shell interpreter without first downloading to a file. The command `curl --proto '=https' --tlsv1.2 ... https://sh.rustup.rs | sh -s -- --default-toolchain none -y` fetches and immediately executes a remote script. If the remote server or network path is compromised, arbitrary code executes on the runner.

Locations:

- `action.yml:70`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three findings in action.yml:

1. script-injection: Moved all ${{ }} expressions from run: blocks into env: blocks. Shell scripts now reference them as plain environment variables (RUNNER_OS, RUNNER_ARCH, RUNNER_TEMP, PARSED_TOOLCHAIN, FLAGS_TARGETS, FLAGS_COMPONENTS, FLAGS_DOWNGRADE). The runner.os/runner.arch/runner.temp context values are accessed via the built-in $RUNNER_OS, $RUNNER_ARCH, $RUNNER_TEMP environment variables that GitHub Actions automatically provides.

2. github-env-injection: All values written to $GITHUB_OUTPUT and $GITHUB_ENV are now sanitized using 'safe=$(printf \'%s\' "$val" | tr -d \'\\n\\r\')' before being written. This applies to the parse step (toolchain output), the flags step (targets and components outputs), and the CARGO_HOME step (CARGO_HOME env var).

3. unsafe-shell: Replaced 'curl ... https://sh.rustup.rs | sh -s -- ...' with a two-step approach: first download to /tmp/rustup-init.sh, then execute 'sh /tmp/rustup-init.sh', then clean up the file. The Windows installer was already downloading to a file before executing, so only the Linux/macOS path needed fixing.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings in action.yml:

1. Flags step (lines 65, 68): Replaced unquoted `${targets//,/ }` and `${components//,/ }` in for-loops with `IFS=,; read -ra arr <<< "$targets"` / `IFS=,; read -ra arr <<< "$components"` patterns, iterating over `"${arr[@]}"` with proper quoting. This prevents shell metacharacter injection from user-controlled inputs.

2. Rustup install step (line 114): Replaced unquoted `$FLAGS_TARGETS$FLAGS_COMPONENTS$FLAGS_DOWNGRADE` concatenated directly into the rustup command with a safe array-based approach. Each flag variable is conditionally split into an array using `read -ra` (guarded by `[[ -n "$VAR" ]]` to avoid empty-element issues), then expanded with `"${target_args[@]}"`, `"${component_args[@]}"`, and `"${downgrade_args[@]}"`. This ensures proper word splitting without allowing shell metacharacter injection.

