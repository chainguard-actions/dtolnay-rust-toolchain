# Hardening Report: dtolnay--rust-toolchain/v1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **dtolnay--rust-toolchain/v1** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

The 'flags' step directly interpolates the attacker-controlled expression `${{inputs.components}}` inside a `run:` block. Specifically, the line `echo "downgrade=${{steps.parse.outputs.toolchain == 'nightly' && inputs.components && ' --allow-downgrade' || ''}}" >> $GITHUB_OUTPUT` embeds `inputs.components` directly in the shell command string rather than routing it through an `env:` variable first.

Locations:

- `action.yml:62`

### github-env-injection (severity: high)

Two steps write attacker-controlled `inputs.*` values to `$GITHUB_OUTPUT` via env vars without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). (1) The 'parse' step maps `inputs.toolchain` to the env var `toolchain` and then writes `echo "toolchain=$toolchain" >> $GITHUB_OUTPUT` (and similar variants) without sanitizing newlines — a newline in the toolchain input could inject arbitrary entries into GITHUB_OUTPUT. (2) The 'flags' step maps `inputs.targets`, `inputs.target`, and `inputs.components` to env vars and writes them to `$GITHUB_OUTPUT` via `echo "targets=..." >> $GITHUB_OUTPUT` and `echo "components=..." >> $GITHUB_OUTPUT` without sanitization.

Locations:

- `action.yml:42`
- `action.yml:44`
- `action.yml:47`
- `action.yml:49`
- `action.yml:51`
- `action.yml:60`
- `action.yml:61`

### unsafe-shell (severity: high)

A `run:` block pipes the output of `curl` directly to `sh` to install rustup: `curl --proto '=https' --tlsv1.2 --retry 10 --retry-connrefused --location --silent --show-error --fail https://sh.rustup.rs | sh -s -- --default-toolchain none -y`. Even though TLS is enforced, piping remote content directly to a shell is an unsafe pattern — if the remote server or the network path is compromised, arbitrary code executes on the runner without any integrity check.

Locations:

- `action.yml:68`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unsafe-shell

**Notes:**

Fixed all three high-severity findings in action.yml: (1) script-injection: replaced direct `${{inputs.components}}` interpolation in the 'flags' step's downgrade line with a shell conditional using the already-env-mapped `$components` variable; (2) github-env-injection: added `printf '%s' "$var" | tr -d '\n\r'` sanitization before every write to $GITHUB_OUTPUT in both the 'parse' and 'flags' steps; (3) unsafe-shell: replaced `curl ... | sh -s --` with a two-step approach that downloads the rustup installer to /tmp/rustup-init.sh, executes it separately, then removes it.

