# GitHub Codespaces burst compute

## Operating model

The recommended default is a **disposable worker**. The VPS checkout stays authoritative; the CLI snapshots the current tree, creates a Codespace from a tiny launcher repository, transfers the snapshot, runs one combined build/test/lint command under `/workspaces/<project>`, retrieves logs/artifacts, and deletes the worker. This prevents stale source trees, cross-agent races and abandoned compute. Use persistent Codespaces only for interactive work, not as shared mutable build workers.

Project-specific skills select commands and own preview deployment health checks, test credentials, ports and database orchestration. This CLI only transports a snapshot, runs a command, retrieves output and cleans up.

## Billing notes (verify against each account plan)

GitHub currently lists 15 GB-month storage + 120 compute hours for Free personal accounts and 20 GB-month + 180 hours for Pro; current published rates list $0.36 per hour for 4-core Codespaces and $0.07 per GB-month storage. Organizational/enterprise billing may differ. These allowances reset per billing cycle; they are not necessarily a one-time first month of free service.

32 GB is a Codespaces volume/maximum selection, not a flat charge for all 32 GB regardless of usage. Storage is measured by actual used volume over time (GB-hours). The base container created from GitHub's default Linux image is excluded from billable storage. A custom base image adds storage usage; a devcontainer configuration that only adds features to the default image can keep the base container excluded. The existing `DYung26/codespaces` launcher uses a custom image and several installation/startup hooks, which makes it a poor burst launcher.

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

- `gh-26` → `~/.config/gh-dyung26`
- `gh-06` → `~/.config/gh-dyung06`
- `gh-df` → `~/.config/gh-dyungfirm`
- `gh-odo` → `~/.config/gh-oyedanolu`
- `gh-do` → `~/.config/gh-danieloyekunle`

The connector authenticated as `DYung26` and confirmed `DYung26/codespaces` exists. The five local VPS CLI profiles and browser profiles could not be verified while the current VPS shell/browser MCP endpoints were returning internal errors.

## References

- https://docs.github.com/en/billing/concepts/product-billing/github-codespaces
- https://docs.github.com/en/codespaces/troubleshooting/troubleshooting-included-usage
- https://docs.github.com/en/codespaces/about-codespaces/understanding-the-codespace-lifecycle
- https://cli.github.com/manual/gh_codespace_create
