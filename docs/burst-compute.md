# GitHub Codespaces burst compute

## Operating model

The recommended default is a **disposable worker**. The VPS checkout stays authoritative; the CLI snapshots the current tree, creates a Codespace from a tiny launcher repository, transfers the snapshot, runs one combined build/test/lint command under `/workspaces/<project>`, retrieves logs/artifacts, and deletes the worker. This prevents stale source trees, cross-agent races and abandoned compute. Use persistent Codespaces only for interactive work, not as shared mutable build workers.

Project-specific skills select commands and own preview deployment health checks, test credentials, ports and database orchestration. This CLI only transports a snapshot, runs a command, retrieves output and cleans up.

## Billing notes (verify against each account plan)

GitHub currently lists 15 GB-month storage + 120 compute hours for Free personal accounts and 20 GB-month + 180 hours for Pro; current published rates list $0.36 per wall-clock hour for a 4-core Codespace and $0.07 per GB-month of billable storage. A 4-core hour consumes 4 included compute-hours, so 120 included hours on Free equate to 30 wall-clock hours on a 4-core worker (or 45 hours on Pro's 180-hour allowance). Organizational/enterprise billing may differ. These allowances reset per billing cycle; they are not necessarily a one-time first month of free service.

The `standardLinux32gb` machine provides 4 cores, 16 GB RAM and a 32 GB disk. The 32 GB disk figure is not a flat monthly charge simply because that machine type was selected: GitHub measures Codespaces storage over time in GB-hours, with the default base image excluded. Storage usage depends on the files and data in the Codespace and its base image. The base container created from GitHub's default Linux image is excluded from billable storage. A custom base image adds storage usage; a devcontainer configuration that only adds features to the default image can keep the base container excluded. The existing `DYung26/codespaces` launcher uses a custom image and several installation/startup hooks, which makes it a poor burst launcher.

Only running Codespaces incur compute charges. Stopped Codespaces incur storage charges until deleted. Closing a browser tab does not stop a Codespace. Inactive Codespaces are deleted after their retention period (default 30 days); the CLI explicitly deletes its worker immediately by default.

## Cost controls

1. Combine checks into one command per worker, so create/setup overhead is amortized.
2. Use a short idle timeout, and explicitly stop and delete workers.
3. Avoid prebuilds unless measurements justify their Actions/storage cost.
4. Exclude `.git`, env files, dependency directories, local build output and `.devcontainer` by default.
5. Review metered usage and set spending budgets/alerts on every account that may own Codespaces.
6. Keep the launcher small, without a custom-image devcontainer, Docker daemon or persistent service setup.
7. Use account aliases to select the right billing/account context intentionally; don't distribute usage simply to evade quotas.

## CLI, skill, or MCP?

Start with a CLI plus skill. The CLI is deterministic and easy to test: account selection, create, transfer, execute, collect and cleanup. The skill teaches the agent when to invoke the CLI and leaves per-project build/deploy details in project-specific skills. Add MCP only if asynchronous multi-job status, cancellation, structured artifacts, concurrency limits or centrally enforced policy become necessary. A later MCP should expose narrow methods (`submit_job`, `get_job_status`, `get_logs`, `get_artifacts`, `cancel_job`) rather than arbitrary shell access, and should persist job state, enforce timeouts and guarantee cleanup.

## Existing launcher repository

The existing `DYung26/codespaces` repo has `.devcontainer/devcontainer.json` with base image `mcr.microsoft.com/devcontainers/javascript-node:20-trixie`, Docker outside-of-Docker, SSHD and startup hooks for Syncthing, cloudflared and MCP services. Do not remove those scripts or modify main branch blindly: that would break your interactive setup.

The create API automatically selects `.devcontainer/devcontainer.json` if present. Therefore the burst path uses a dedicated launcher repository `DYung26/codespaces-burst` with no `.devcontainer` directory (default Linux image). The worker then unpacks the current source snapshot into `/workspaces/<project>` to match the local `~/Projects/* → /workspaces/*` convention. Keep custom environment scripts separately; they can remain in the original repo and in dotfiles.

## Account aliases found in dotfiles

- `gh-26` → `~/.config/gh` (the active DYung26 profile; the old dedicated `gh-dyung26` directory had an invalid token)
- `gh-06` → `~/.config/gh-dyung06`
- `gh-df` → `~/.config/gh-dyungfirm`
- `gh-odo` → `~/.config/gh-oyedanolu`
- `gh-do` → `~/.config/gh-danieloyekunle`

The connector authenticated as `DYung26` and confirmed `DYung26/codespaces` exists. The four secondary VPS CLI profiles currently have invalid tokens and require reauthentication. `gh-26` now points to the valid active DYung26 profile.

## References

- https://docs.github.com/en/billing/concepts/product-billing/github-codespaces
- https://docs.github.com/en/codespaces/troubleshooting/troubleshooting-included-usage
- https://docs.github.com/en/codespaces/about-codespaces/understanding-the-codespace-lifecycle
- https://cli.github.com/manual/gh_codespace_create
