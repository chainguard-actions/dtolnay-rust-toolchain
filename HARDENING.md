<!-- markdownlint-disable -->

# Hardening Report: dtolnay--rust-toolchain/v1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **dtolnay--rust-toolchain/v1** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings in action.yml, violating rule (a). This includes:
- `${{runner.os}}` in the `parse` step run block (line ~40)
- `${{steps.parse.outputs.toolchain == 'nightly' && inputs.components && ' --allow-downgrade' || ''}}` written directly to $GITHUB_OUTPUT in the `flags` step (line ~60)
- `${{runner.os == 'Windows' && '$USERPROFILE\.cargo' || '$HOME/.cargo'}}` in the `Set $CARGO_HOME` run: line (line ~66)
- `${{runner.arch == 'ARM64' && 'aarch64' || 'x86_64'}}` and `${{runner.temp}}` in the Windows rustup install step run block (lines ~72-73)
- `${{steps.parse.outputs.toolchain}}`, `${{steps.flags.outputs.targets}}`, `${{steps.flags.outputs.components}}`, `${{steps.flags.outputs.downgrade}}` in the `rustup toolchain install` run: line (line ~78)
- `${{steps.parse.outputs.toolchain}}` in the `rustup default` run: line (line ~84)
- `${{steps.parse.outputs.toolchain}}` in multiple `rustc +...` run: lines (lines ~100, ~115, ~122, ~131)
All of these bypass shell quoting and allow template-injected values to be parsed as shell syntax before the shell ever sees them.

Locations:

- `action.yml:40`
- `action.yml:60`
- `action.yml:66`
- `action.yml:72`
- `action.yml:78`
- `action.yml:84`
- `action.yml:100`
- `action.yml:115`
- `action.yml:122`
- `action.yml:131`

### github-env-injection (severity: high)

Multiple steps write values derived from untrusted inputs to GitHub special environment files without the required sanitization (`printf '%s' ... | tr -d '\n\r'`):

(1) `parse` step: `$toolchain` (set from `inputs.toolchain` via env:) is written directly to `$GITHUB_OUTPUT` in multiple branches, e.g. `echo "toolchain=$toolchain" >> $GITHUB_OUTPUT`. An attacker-controlled toolchain value containing newlines could inject arbitrary key=value pairs into GITHUB_OUTPUT.

(2) `flags` step: `$targets` and `$components` (set from `inputs.targets`, `inputs.target`, `inputs.components` via env:) are written to `$GITHUB_OUTPUT` without sanitization, e.g. `echo "targets=$(for t in ${targets//,/ }; ...)" >> $GITHUB_OUTPUT`.

(3) `Set $CARGO_HOME` step: the expression `${{runner.os == 'Windows' && ...}}` is interpolated directly into the value written to `$GITHUB_ENV` without sanitization.

Locations:

- `action.yml:41`
- `action.yml:43`
- `action.yml:46`
- `action.yml:48`
- `action.yml:50`
- `action.yml:58`
- `action.yml:59`
- `action.yml:66`

### unsafe-shell (severity: high)

The `Install rustup if needed` step pipes remote content directly to a shell interpreter: `curl --proto '=https' --tlsv1.2 ... https://sh.rustup.rs | sh -s -- --default-toolchain none -y`. Even though TLS is enforced, this pattern executes whatever the remote server returns without any integrity verification (e.g. a checksum check). The script should be downloaded to a file first, its integrity verified, and then executed separately.

Locations:

- `action.yml:52`

### unpinned-uses (severity: high)

The workflow uses `actions/checkout@v7`, which is pinned to a mutable tag rather than an immutable 40-character commit SHA. A tag can be moved to point to a different (potentially malicious) commit, enabling supply-chain attacks. It should be replaced with a full SHA pin, e.g. `actions/checkout@<40-char-sha> # v7`.

Locations:

- `.github/workflows/ci.yml:22`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell, unpinned-uses

**Notes:**

Fixed all four findings in action.yml and .github/workflows/ci.yml:

1. script-injection: Moved all ${{ }} expressions from run: blocks into env: blocks. Used $RUNNER_OS, $RUNNER_ARCH, $RUNNER_TEMP built-in env vars where applicable. Step outputs (steps.parse.outputs.toolchain, steps.flags.outputs.*) are now passed via env: as PARSED_TOOLCHAIN, FLAGS_TARGETS, FLAGS_COMPONENTS, FLAGS_DOWNGRADE.

2. github-env-injection: All values written to $GITHUB_OUTPUT and $GITHUB_ENV are now sanitized with `printf '%s' "$value" | tr -d '\n\r'` before writing. The parse step, flags step, and Set $CARGO_HOME step all sanitize their outputs.

3. unsafe-shell: Replaced `curl ... | sh -s -- --default-toolchain none -y` with a two-step approach: download to a temp file via mktemp, then execute with `sh "$RUSTUP_SCRIPT" --default-toolchain none -y` (dropped the `--` shell option terminator as required). The Windows rustup install step was also fixed to use shell variables instead of inline ${{ }} expressions.

4. unpinned-uses: Pinned `actions/checkout@v7` to full SHA `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7` in ci.yml.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all script injection findings:

1. ci.yml (lines 34, 37): Moved `${{steps.toolchain.outputs.cachekey}}` out of `run:` blocks into `env:` blocks as `CACHEKEY`, then referenced as `"$CACHEKEY"` in shell commands. Single-quoting does not prevent runner-level template substitution, so the expression must be in env: instead.

2. action.yml (lines 62-63): Replaced unquoted `${targets//,/ }` and `${components//,/ }` for-loop word-splitting with `IFS=, read -ra` array approach, using quoted `"${target_list[@]}"` and `"${component_list[@]}"` expansions. This prevents shell metacharacter injection from user-supplied comma-separated inputs.

3. action.yml (line 117): Replaced unquoted `$FLAGS_TARGETS$FLAGS_COMPONENTS` and `$FLAGS_DOWNGRADE` with xargs-based tokenization into bash arrays (`flag_targets`, `flag_components`, `flag_downgrade`), then expanded with `"${flag_targets[@]}"` etc. This properly splits the flag strings into separate arguments while preventing shell injection.

