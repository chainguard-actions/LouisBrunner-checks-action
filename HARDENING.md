<!-- markdownlint-disable -->

# Hardening Report: LouisBrunner--checks-action/v3.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **LouisBrunner--checks-action/v3.1.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions using mutable version tags instead of pinned 40-character SHA digests, making them vulnerable to supply-chain attacks if the tag is moved.

- build.yml: `actions/checkout@v6`, `oven-sh/setup-bun@v2`
- examples.yml: `actions/checkout@v6` (used in every job)
- lint-gha.yml: `actions/checkout@v6`, `ibiqlik/action-yamllint@v3`
- update-tags.yaml: `actions/checkout@v6`, `rickstaa/action-create-tag@v1`

Locations:

- `.github/workflows/build.yml:11`
- `.github/workflows/build.yml:13`
- `.github/workflows/examples.yml:10`
- `.github/workflows/lint-gha.yml:16`
- `.github/workflows/lint-gha.yml:17`
- `.github/workflows/update-tags.yaml:11`
- `.github/workflows/update-tags.yaml:20`

### permissions (severity: medium)

None of the workflow files define a top-level `permissions:` key, and no individual jobs define job-level `permissions:` keys. Without explicit permissions, workflows run with the default (potentially broad) token permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/build.yml:1`
- `.github/workflows/examples.yml:1`
- `.github/workflows/lint-gha.yml:1`
- `.github/workflows/update-tags.yaml:1`

### script-injection (severity: high)

Sub-rule (a): In update-tags.yaml, the expression `${{ github.ref_name }}` is interpolated directly inside a `run:` shell command string. Before the shell executes the command, GitHub Actions substitutes the raw value of `github.ref_name` into the script text, allowing a specially crafted tag name (e.g. containing shell metacharacters) to inject arbitrary commands.

Offending line:
  `TRUNC_VER=$(echo ${{ github.ref_name }} | cut -d '.' -f 1)`

Locations:

- `.github/workflows/update-tags.yaml:17`

### github-env-injection (severity: high)

In update-tags.yaml, the value of `${{ github.ref_name }}` is captured into the shell variable `TRUNC_VER` (via direct expression interpolation in the `run:` block) and then written to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A tag name containing newline characters could inject additional key=value pairs into the GitHub output context.

Offending lines:
  `TRUNC_VER=$(echo ${{ github.ref_name }} | cut -d '.' -f 1)`
  `echo "tag=$TRUNC_VER" >> $GITHUB_OUTPUT`

Locations:

- `.github/workflows/update-tags.yaml:19`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings across four workflow files:

1. **unpinned-uses**: Pinned all action references to full 40-char SHAs with tag comments:
   - actions/checkout@v6 → @d23441a48e516b6c34aea4fa41551a30e30af803 (all files)
   - oven-sh/setup-bun@v2 → @0c5077e51419868618aeaa5fe8019c62421857d6 (build.yml)
   - ibiqlik/action-yamllint@v3 → @2576378a8e339169678f9939646ee3ee325e845c (lint-gha.yml)
   - rickstaa/action-create-tag@v1 → @a1c7777fcb2fee4f19b0f283ba888afa11678b72 (update-tags.yaml)

2. **permissions**: Added top-level `permissions: {}` to all four workflow files. Added job-level `permissions: { checks: write }` to all jobs in examples.yml (needed for the checks API), and `permissions: { contents: write }` to the retag job in update-tags.yaml (needed to push tags).

3. **script-injection**: In update-tags.yaml, moved `${{ github.ref_name }}` from the run: shell string into the step's env: block as REF_NAME, then referenced it safely as "$REF_NAME" in the shell script.

4. **github-env-injection**: In update-tags.yaml, sanitized the value before writing to $GITHUB_OUTPUT using `printf '%s' "$REF_NAME" | tr -d '\n\r'` to strip newlines that could inject additional key=value pairs.

