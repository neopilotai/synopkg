# Project Analysis and Improvement Plan

## Executive Summary

Synopkg has a broad Rust test suite and established CI for compilation, formatting, linting, and tests. The clearest near-term gains are to keep architecture/release metadata current, validate the npm artifacts users install, and improve configuration diagnostics. This is a repository-level assessment, not a security or performance audit.

## Current State

- **Core design:** The CLI follows an analyse, inspect, and run pipeline. `Context` owns the loaded `Disk`, `Instance` values, `Sources`, and version groups.
- **Test baseline:** `cargo test --locked` passes: 706 tests across the binary and integration targets.
- **CI:** Pull requests and pushes run Cargo check, formatting, Clippy, and tests on the configured build matrix. The site has a separate build-and-deploy workflow.
- **Release:** Release builds publish platform-specific npm binaries, then publish the npm wrapper package.
- **Known maintenance signals:** `.notes/context.md` describes an older `Context` shape. The CI and benchmark workflows still use the label “v14” while Cargo and npm metadata report version 15.0.0.
- **Known implementation limitation:** `Context::create` converts version-group and dependency-type validation errors into one-element error lists; nearby TODOs call out returning every error.

## Prioritized Plan

### 1. Validate Published npm Artifacts

**Why:** Rust tests validate the implementation, but they do not establish that a published npm package selects and launches the correct binary on each supported platform.

**Work:** Add a release or scheduled smoke check that installs the generated package artifact and runs a harmless command such as `synopkg --version` or `synopkg --help` on representative matrix targets. Keep this separate from publishing credentials and production release side effects.

**Acceptance:** Each supported target installs its generated package and the command exits successfully; failures identify the target and artifact.

### 2. Refresh Architecture and Version Metadata

**Why:** Stale architectural notes slow onboarding, and workflow names currently suggest v14 despite the v15 package version.

**Work:** Update `.notes/context.md` to match the current `Context`, `Disk`, and `Sources` ownership model. Rename the “v14” CI and benchmark workflow labels if they no longer describe their purpose; keep any historical migration references intact.

**Acceptance:** Architecture notes match the current types and pipeline; workflow labels agree with the supported release line or use version-neutral names.

### 3. Return Multiple Configuration Errors

**Why:** Invalid selectors and version-group configuration currently surface one error at a time, requiring repeated edit-and-run cycles.

**Work:** Refactor validation boundaries in `Context::create` and `Rcfile` to collect independent errors where possible, while preserving existing error wording and exit behavior for a single error.

**Acceptance:** A fixture with two independent invalid selectors reports both; existing configuration tests and CLI output tests pass.

### 4. Make Performance Checks Decision-Useful

**Why:** The existing benchmark measures `synopkg list` on the Fluid Framework fixture, but provides no regression threshold and covers only one command path.

**Work:** First capture repeatable baselines for representative read-only and registry-backed workflows. Add a threshold only after measuring normal run-to-run variance; avoid turning noisy benchmarks into a flaky merge gate.

**Acceptance:** Benchmark output is reproducible enough to compare revisions, documents the command and fixture, and reports regressions against a justified threshold.

## Suggested Sequence

1. Refresh `.notes/context.md` and remove or correct stale workflow labels.
2. Add npm artifact smoke coverage, starting with the lowest-cost supported targets.
3. Improve multi-error configuration reporting with focused regression tests.
4. Expand benchmark coverage and decide whether any threshold is stable enough for CI.

## Verification Baseline

Run `cargo test --locked`, `just check-cargo`, `just check-formatting`, and `just check-clippy` for Rust changes. Run `pnpm --dir site build` for site changes. Run the artifact smoke check against generated package artifacts before relying on it in release automation.

## Evidence

- Architecture overview: `.notes/context.md`
- Pipeline and dispatch: `src/synopkg.rs`
- Context construction and validation: `src/context.rs`
- Pull request checks: `.github/workflows/ci.yaml`
- Cross-platform npm release flow: `.github/workflows/release.yaml`
- Site build: `.github/workflows/deploy-site.yaml`
- Current benchmark: `.github/workflows/benchmark.yaml` and `justfile`