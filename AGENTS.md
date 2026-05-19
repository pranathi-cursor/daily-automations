# AGENTS.md

## Repository overview

This is a **documentation/notes repository** for an account management team. It contains:

- `weeks/` — weekly markdown files with meeting notes, action items, and customer tracking
- `.cursor/skills/account-summary-refresh/SKILL.md` — a Cursor AI skill definition that orchestrates MCP servers (Notion, Slack, Databricks) to refresh account summaries in the Book of Business Notion view

There is **no application code**, no package manager, no build system, no test framework, and no services to run.

## Cursor Cloud specific instructions

- **`gws` (Google Workspace CLI)** v0.22.5. The real binary lives at `/usr/local/bin/gws-real`. `/usr/local/bin/gws` is a thin wrapper that refreshes the OAuth access token before each invocation. Usage: `gws <service> <resource> <method> [flags]`. Run `gws --help` for details.
- **`gws` authentication:** The update script reads `client_id`, `client_secret`, and `refresh_token` from `$GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` (written from the `GWS_CREDENTIALS_JSON` secret), exchanges them for a short-lived access token via `https://oauth2.googleapis.com/token`, and exports `GOOGLE_WORKSPACE_CLI_TOKEN`. No manual `gws auth login` needed. If the token refresh fails, check that the `GWS_CREDENTIALS_JSON` secret contains valid `authorized_user` JSON and that `oauth2.googleapis.com` is in the egress allowlist.
- **No other dependencies to install.** This repo has no `package.json`, `requirements.txt`, `pyproject.toml`, or similar.
- **No services to start.** The only "application" is the Cursor skill (`.cursor/skills/account-summary-refresh/SKILL.md`), which is executed declaratively by the Cursor agent via MCP servers (Notion, Slack, Databricks SQL). It does not run as a standalone process.
- **No lint/test/build commands.** Standard development commands (lint, test, build) are not applicable to this repository.
- **To use the account-summary-refresh skill**, the MCP servers `Notion`, `Slack`, and `Databricks SQL` must be authenticated. The skill is triggered by invoking it through Cursor with phrases like "refresh accounts" or "update book of business". See the skill file for full details.
- **Editing content:** All content files are plain markdown. No special tooling is needed to edit them.
