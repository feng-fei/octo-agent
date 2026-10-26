# fnOS NAS adaptation audit (initial)

Date: 2026-10-26
Target branch: `feat/fnos-nas-agent-base`
Upstream base: `main` (octo-agent)

## Purpose

Use upstream octo-agent as the agent runtime and add a narrowly scoped fnOS NAS management layer. Do not replace the upstream agent loop or broadly rewrite the web UI before proving the baseline builds and runs.

## Current evidence

- The repository is a Go application. The primary CLI entry is `cmd/octo/main.go`; the HTTP server implementation is under `internal/server/`.
- `go.mod` declares `go 1.25.0`; the CI workflow uses Go 1.25.
- The Makefile's introductory comment says “Go 1.22+”, which conflicts with the actual module/toolchain requirement and should be corrected.
- The module path remains `github.com/open-octo/octo-agent`. The source imports and Makefile linker flags also use that path. Keep it unchanged for the first adaptation milestone; changing it would require a repository-wide import and linker-flag migration and adds risk without helping fnOS deployment.
- The CI workflow `.github/workflows/go.yml` runs Go vet/build/test across Linux, macOS, and Windows, plus Svelte checks/tests. The normal Go test command is `go test -race ./...`.
- `COMPATIBILITY.md` describes Linux as a first-class platform and Linux sandboxing via Landlock, but this is not evidence that a particular fnOS release, CPU architecture, filesystem layout, or service manager has been tested.
- `SECURITY.md` defines octo as a single-user agent that can execute arbitrary shell commands. Its server defaults to loopback; exposing it on a LAN requires access-key authentication. This threat model must remain explicit in the NAS package and setup guide.
- `internal/version/version.go` currently reports `1.17.20`; release metadata and CI results must be checked independently before describing this fork build as a tested release.

## Findings and decisions

### P0 — prove the upstream baseline before feature work

1. Run the existing CI workflow against this branch (Go vet, formatting, build, race-enabled tests, Svelte checks/tests).
2. Record failures as upstream baseline failures or fork-introduced failures; do not start a large feature patch until the baseline is understood.
3. Do not change the Go module path in this phase.

### P1 — fnOS packaging and runtime compatibility

1. Record the actual fnOS version, CPU architecture (`x86_64/amd64` or `aarch64/arm64`), libc/runtime constraints, available Go toolchain on the build host, and supported service manager/package hooks.
2. Prefer cross-compiling a static or suitably portable Linux binary in CI/build environment rather than requiring Go on the NAS.
3. Verify the resulting binary with `file`, `ldd` (where applicable), `octo version`, and a smoke test on the actual NAS.
4. Keep application data, logs, config, and backups in explicit persistent directories; never assume a desktop user's home directory or a writable system path.
5. Bind the web server to loopback by default. For LAN access, require explicit opt-in and document authentication and TLS/tunnel options.

### P1 — NAS-specific capabilities (add as a separate, minimal layer)

Implement read-only diagnostics first:
- system and kernel information;
- CPU load, memory/swap;
- disk, filesystem, mount and capacity information;
- service/process status;
- recent system/service logs, with bounded output and secret redaction.

Then add carefully gated actions:
- restart a named service only from an allowlist;
- validate service names and arguments without shell string concatenation;
- require confirmation for disruptive operations;
- audit each action and its result;
- deny arbitrary disk formatting, partitioning, filesystem repair, account changes, and broad destructive commands by default.

Do not treat the general-purpose terminal tool as a substitute for a structured NAS-management API.

## Acceptance gates

- [ ] CI on the adaptation branch passes, or all failures have a documented and reproducible explanation.
- [ ] Build artifact starts on the target fnOS device and reports version/architecture correctly.
- [ ] Web UI is reachable only on the intended interface; unauthenticated LAN access is rejected.
- [ ] Read-only diagnostic endpoints return normalized human-readable units (GiB/MiB, percentages, timestamps) and handle missing utilities/permissions gracefully.
- [ ] Service actions are allowlisted, confirmed, logged, and tested against a non-critical test service.
- [ ] FPK install, upgrade, restart, stop, uninstall, and persistent-data behavior are verified on the actual device.

## Current status

This is an initial source review and plan only. No source code behavior has been changed by this document. No CI run, local test suite, binary build, FPK installation, or on-device test is claimed as completed.
