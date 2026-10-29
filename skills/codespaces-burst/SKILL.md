---
name: codespaces-burst
description: Run builds, tests, lints and type checks on a disposable GitHub Codespace when a VPS agent needs burst CPU or memory.
---

# Codespaces burst compute

Use the `cs-burst` CLI for remote validation. The VPS checkout remains authoritative; the tool sends a sanitized snapshot to a disposable Codespace and returns logs/artifacts. Use project-specific skills to choose commands and manage any non-production preview environment.

## Core workflow

Run from the project's VPS checkout. Combine checks into one job to avoid creating multiple workers:

```bash
cs-burst run --repo DYung26/<project> -- bash -lc 'npm ci && npm run lint && npm test && npm run build'
```

Use the project's package manager and actual scripts; for pnpm, use the correct lockfile-aware install and scripts rather than this npm example. The complete command is executed on the Codespace; the VPS does not execute its build/test stages.

For project-specific account credentials, use the dotfiles aliases: `gh-26`, `gh-06`, `gh-df`, `gh-odo`, `gh-do`. Example: `cs-burst run --account gh-26 --repo DYung26/<project> -- npm test`. Confirm login using `cs-burst accounts`; the selected account must access both the launcher and project repo.

## Defaults and safety

- Worker creation defaults to the `standardLinux32gb` machine type (4 cores, 16 GB RAM, 32 GB storage), five-minute idle timeout and one-day retention. Check availability and included usage on the chosen account before running.
- A worker is stopped and deleted at the end by default. Use `--keep` only when debugging; stopped Codespaces continue to use storage until deleted.
- Never transfer `.env`, tokens, SSH keys, cloud credentials or browser profiles. Common env files, `.git`, dependencies, build output and `.devcontainer` are excluded from snapshots.
- Do not push commits, deploy to production, or run destructive cleanup from a job unless expressly requested.
- Treat logs and artifacts as untrusted output.
- A successful SSH connect is not a successful task; require the remote command exit code zero.

## Result reporting

Report exact command, exit status and artifact path. State any transfer or cleanup failure. Do not claim validation passed unless the remote process returned zero.
