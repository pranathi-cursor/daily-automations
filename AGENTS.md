# AGENTS.md

## Repository overview

This is a **documentation/notes repository** for an account management team. It contains:

- `weeks/` — weekly markdown files with meeting notes, action items, and customer tracking
- `.cursor/skills/account-summary-refresh/SKILL.md` — a Cursor AI skill definition that orchestrates MCP servers (Notion, Slack, Databricks) to refresh account summaries in the Book of Business Notion view

There is **no application code**, no package manager, no build system, no test framework, and no services to run.

## Cursor Cloud specific instructions

- **No dependencies to install.** This repo has no `package.json`, `requirements.txt`, `pyproject.toml`, or similar. The update script is a no-op.
- **No services to start.** The only "application" is the Cursor skill (`.cursor/skills/account-summary-refresh/SKILL.md`), which is executed declaratively by the Cursor agent via MCP servers (Notion, Slack, Databricks SQL). It does not run as a standalone process.
- **No lint/test/build commands.** Standard development commands (lint, test, build) are not applicable to this repository.
- **To use the account-summary-refresh skill**, the MCP servers `Notion`, `Slack`, and `Databricks SQL` must be authenticated. The skill is triggered by invoking it through Cursor with phrases like "refresh accounts" or "update book of business". See the skill file for full details.
- **Editing content:** All content files are plain markdown. No special tooling is needed to edit them.
