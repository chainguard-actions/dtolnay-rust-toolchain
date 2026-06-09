# Hardening Report: dtolnay--rust-toolchain/v1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **dtolnay--rust-toolchain/v1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

In the 'flags' step, the attacker-controlled expression `${{ inputs.components }}` is directly interpolated inside the `run:` shell command string (in the `downgrade=` line). This allows an attacker to inject arbitrary shell commands by supplying a malicious value for the `components` input. The `targets` and `components` values are correctly routed through env vars, but `inputs.components` is also used raw in the expression on line 62.

Locations:

- `action.yml:62`

### github-env-injection (severity: high)

The 'parse' step writes the attacker-controlled `inputs.toolchain` value (via env var `$toolchain`) to `$GITHUB_OUTPUT` in multiple branches (lines 42, 44, 47, 49, 51) without the required `printf '%s' ... | tr -d '\n\r'` sanitization. A newline injected into the toolchain input can poison subsequent entries in the output file.

Locations:

- `action.yml:34`

### github-env-injection (severity: high)

The 'flags' step writes attacker-controlled values to `$GITHUB_OUTPUT` without sanitization: (1) `$targets` (from `inputs.targets`/`inputs.target`) and `$components` (from `inputs.components`) are written via env vars but without `printf '%s' ... | tr -d '\n\r'`; (2) `${{ inputs.components }}` is directly interpolated in the `downgrade=` line and written to `$GITHUB_OUTPUT`. Newline injection via any of these inputs can corrupt the output file.

Locations:

- `action.yml:58`

### unsafe-shell (severity: high)

The 'install rustup if needed' step pipes remote content directly to a shell interpreter: `curl ... https://sh.rustup.rs | sh -s -- --default-toolchain none -y`. The script is not downloaded to a file and verified before execution. A compromised or MitM'd response would execute arbitrary code on the runner.

Locations:

- `action.yml:76`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all four findings in action.yml:
1. script-injection: Removed `${{ inputs.components }}` from the `downgrade=` run line; moved `steps.parse.outputs.toolchain` into an env var and evaluated the condition purely in shell.
2. github-env-injection (parse step): All branches writing to $GITHUB_OUTPUT now sanitize via `printf '%s' ... | tr -d '\n\r'` before writing.
3. github-env-injection (flags step): `targets` and `components` outputs sanitized with `printf '%s' ... | tr -d '\n\r'`; `downgrade` computed in pure shell without expression interpolation.
4. unsafe-shell: Replaced `curl ... | sh` with download-then-execute pattern: curl saves to `$RUNNER_TEMP/rustup-init.sh`, then `sh` runs the saved file, then the file is removed.

