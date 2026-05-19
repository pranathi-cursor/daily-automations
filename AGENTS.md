# AGENTS.md

## Repository overview

This is a **documentation/notes repository** for an account management team. It contains:

- `weeks/` — weekly markdown files with meeting notes, action items, and customer tracking
- `.cursor/skills/account-summary-refresh/SKILL.md` — a Cursor AI skill definition that orchestrates MCP servers (Notion, Slack, Databricks) to refresh account summaries in the Book of Business Notion view

There is **no application code**, no package manager, no build system, no test framework, and no services to run.

## Cursor Cloud specific instructions

- **`gws` (Google Workspace CLI)** is installed at `/usr/local/bin/gws` (v0.22.5). The update script installs the binary and writes credentials from secrets on every VM startup. Usage: `gws <service> <resource> <method> [flags]`. Supports Drive, Gmail, Calendar, Sheets, Docs, Chat, Admin, and more. Run `gws --help` for details.
- **`gws` authentication:** Handled automatically by the update script using two secrets: `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` (path where credentials are written) and `GWS_CREDENTIALS_JSON` (the credential JSON content). No manual `gws auth login` needed when these secrets are set.
- **`gws` TLS caveat:** The gws Rust binary may fail with `tls handshake eof` in Cloud Agent VMs due to TLS stack incompatibilities. The credentials themselves are valid (verifiable via `curl` or Python `urllib` token refresh against `oauth2.googleapis.com`). If gws calls fail, ensure `oauth2.googleapis.com`, `www.googleapis.com`, and `gmail.googleapis.com` are on the egress allowlist (see the SKILL.md "Egress allowlist" section).
- **No other dependencies to install.** This repo has no `package.json`, `requirements.txt`, `pyproject.toml`, or similar.
- **No services to start.** The only "application" is the Cursor skill (`.cursor/skills/account-summary-refresh/SKILL.md`), which is executed declaratively by the Cursor agent via MCP servers (Notion, Slack, Databricks SQL). It does not run as a standalone process.
- **No lint/test/build commands.** Standard development commands (lint, test, build) are not applicable to this repository.
- **To use the account-summary-refresh skill**, the MCP servers `Notion`, `Slack`, and `Databricks SQL` must be authenticated. The skill is triggered by invoking it through Cursor with phrases like "refresh accounts" or "update book of business". See the skill file for full details.
- **Editing content:** All content files are plain markdown. No special tooling is needed to edit them.
