---
name: account-summary-refresh
description: Refresh the Summary and Up-to-date status on the Book of Business Notion view by synthesizing the last 14 days of Gong calls, Notion meeting notes, Slack messages (channels + DMs), and Salesforce activity for each account. Writes a dated Account Plan note with the deeper bullets. Use when the user says "refresh accounts", "update book of business", "refresh account summaries", "weekly account refresh", or wants any account row in the Book of Business resynced from recent signal.
---

# Account Summary Refresh

Refreshes the **Book of Business** Notion view (`https://www.notion.so/cursorai/2f4da74ef045810eb479f860344b6f3b`) by pulling the last 14 days of customer signal for each row and writing back a 1-line Summary + a dated Account Plan note. Designed to be invokable ad hoc and also wrapped in a weekly cron automation.

## Required MCP servers (must be authenticated)

- `dashboard-team-1-Notion` — read view + write Summary/checkbox + create Account Plan pages
- `dashboard-team-1-Databricks SQL` — Gong calls + Salesforce mirror
- `dashboard-team-1-Slack` — channel reads, search, DMs

If any of these only expose `mcp_auth`, call the auth tool and ask the user to accept the prompt before continuing.

## Canonical IDs (do not re-derive)

```yaml
notion:
  book_of_business_data_source: collection://2f4da74e-f045-815f-acce-000b4c32228b
  account_plans_data_source:    collection://2f4da74e-f045-81a0-a7f6-000b2d509206
databricks:
  gong:
    calls:             dev.rperry.calls
    speaker_map:       dev.rperry.speaker_map
    transcript_chunks: dev.rperry.transcript_chunks
    utterances:        dev.rperry.utterances
    call_extensive:    dev.rperry.call_extensive
  salesforce:
    account:     revops.pt_salesforce.account
    opportunity: revops.pt_salesforce.opportunity
slack:
  internal_workspace_domain: anysphere.slack.com
  enterprise_workspace_domain: anysphere.enterprise.slack.com
```

## Notion column shape (Book of Business)

| Column | Type | Write? | Notes |
|---|---|---|---|
| Name | title | no | Account name (e.g. "Benchling", "Elastic", "EToro") |
| ADM | person | no | Account Development Manager — NOT the AE |
| Band | select (T1/T2/T3/T4) | no | Tier |
| Renewal | date | no | |
| Internal Slack | url | no | Account's primary internal Slack channel (sometimes null) |
| **Summary** | text | **yes** | 1 terse sentence, matching existing tone |
| **Up to date** | checkbox (`__YES__` / `__NO__`) | **yes** | True if any call in last 14 days |
| Meetings | relation | no | |
| Account Plan | relation | no | Link to a new Account Plan page we create |
| Notes & Assets | relation | no | |
| Task | relation | no | |
| p0 CC, p0 EBR | checkbox | no | |

## Inputs

- `--accounts <name|all>` — defaults to `all` (every row in the view). Can pass a single account name (e.g. `Benchling`) or comma-separated list.
- `--window-days <N>` — defaults to `14`.
- `--dry-run` — print the proposed updates without writing to Notion.

If invoked with no args, refresh **all** accounts in the Book of Business view.

## Workflow

### 1. Pull the Book of Business rows

```sql
-- via notion-query-data-sources, SQL mode
SELECT Name, Band, "Up to date", Summary, "Internal Slack", url
FROM "collection://2f4da74e-f045-815f-acce-000b4c32228b"
ORDER BY Band, Name
```

For each row, run steps 2–7. Process accounts sequentially (Notion writes are cheap; LLM synthesis dominates cost).

### 2. Look up the AE from Salesforce

```sql
SELECT Name, Owner_Name__c, Owner_Cursor_Email__c, Owner_Role_Name__c,
       Technical_Account_Manager__c, Renewal_Manager__c, Most_Recent_AE_Activity__c
FROM revops.pt_salesforce.account
WHERE Name ILIKE '<account_name>'
LIMIT 5
```

If multiple accounts match (e.g. "Elastic", "ElasticRun", "ElasticDevs"), prefer the one with an assigned AE (`Owner_Name__c` not in `('Account Pool', NULL)`). Capture AE name + cursor email — used downstream for Slack searches.

Also pull open opportunities:

```sql
SELECT Name, StageName, Amount, CloseDate, NextStep, Description
FROM revops.pt_salesforce.opportunity
WHERE AccountId = (SELECT Id FROM revops.pt_salesforce.account WHERE Name = '<account_name>' LIMIT 1)
  AND IsClosed = false
ORDER BY CloseDate ASC LIMIT 10
```

### 3a. Detect off-Gong calls from Slack mentions (do this BEFORE the Gong query)

**Critical:** not all customer calls are in Gong. When the customer hosts on **their** Zoom / Google Meet / Webex, the call exists but `dev.rperry.calls` will not have it. Example: the 2026-05-12 discovery call with Brent Newton (Elastic events lead) was on Elastic's Zoom — Pranathi said *"I'll have the transcript. It's on their zoom"* — and there is no Gong record of it.

Scan the first-pass Slack search results for phrases that imply an off-Gong call happened:
- `"the call with <name>"`
- `"on their zoom"`, `"their zoom"`, `"their meet"`, `"on their teams"`
- `"call w/ <name>"`, `"synced with"`, `"sync with"`
- `"discovery call"`, `"intro call"` (when followed by debriefing language and there's no Gong match)
- Calendly links from external domains (e.g. `calendly.com/brent-newton-elastic`)

For each detected off-Gong call, capture:
- Approximate date/time (from the Slack message timestamp)
- Attendees mentioned in the message
- Source link (Slack permalink) and the literal phrase that attested to it

In the Account Plan note, list these alongside Gong calls under "Last 14 days at a glance" with source `Off-Gong (Slack-attested)` so the count is accurate.

### 3b. Gather Gong signal — CAST A WIDE NET

**Critical lesson from initial design:** filtering only on `speaker_map.email_domain = '<account>.com'` misses:
- In-person calls (speaker_map has the conference-room name, no external email)
- Calls where the external participants weren't recorded as speakers
- Calls where the company uses a domain that doesn't match the account name (e.g. `Kraken Crypto` → `payward.com`)

Use **all three** signals UNION'd:

```sql
-- A) by external speaker email_domain (lowercase company name + .com/.co/.io variants)
WITH by_domain AS (
  SELECT DISTINCT c.call_id
  FROM dev.rperry.calls c JOIN dev.rperry.speaker_map s ON c.call_id = s.call_id
  WHERE s.email_domain IN ('<acct>.com','<acct>.co','<acct>.io')  -- derive from account name
    AND c.started >= date_sub(current_date(), 14)
),
-- B) by call title containing the account name or a known alias
by_title AS (
  SELECT call_id FROM dev.rperry.calls
  WHERE started >= date_sub(current_date(), 14)
    AND (LOWER(title) LIKE '%<account_lower>%' OR <other_aliases>)
),
-- C) by AE participation (catch internal-only prep / strategy calls)
by_ae AS (
  SELECT DISTINCT c.call_id
  FROM dev.rperry.calls c JOIN dev.rperry.speaker_map s ON c.call_id = s.call_id
  WHERE s.email = '<ae_email>'
    AND c.started >= date_sub(current_date(), 14)
    AND (LOWER(c.title) LIKE '%<account_lower>%')
)
SELECT call_id FROM by_domain UNION SELECT call_id FROM by_title UNION SELECT call_id FROM by_ae
```

For each matched call:
- Pull the brief + key_points_json + highlights_json + outline_json from `dev.rperry.call_extensive`
- Pull 30-40 long utterances from `dev.rperry.utterances` joined to `speaker_map` for verbatim quote material

### 4. Gather Slack signal — DMs and channels matter equally, and external contacts matter more than the channel itself

**Critical lessons from initial design (validated against Benchling + Elastic + missed-Brent debugging):**
- The `Internal Slack` URL on the Notion row is **not** the only place activity happens. Internal coordination channels, AE 1:1 DMs, group DMs with externals, and `#team-field-eng` style cross-cutting channels often hold the freshest signal.
- **External contacts are often only reachable via group DMs**, not channels. Example: the Elastic Madrid onsite was negotiated entirely in a 3-person group DM (Brent Newton + Will + Pranathi). A channel-only search missed it completely.
- **First-name searches matter**. Searching Slack for "brent" found Brent Newton; searching for "brent elastic" found nothing because Slack search doesn't AND-match user profiles to message text.

Run all of:

1. **Account's `Internal Slack` channel** (from the Notion row), last 14 days. Use unix timestamp `oldest = now - 14*24*60*60`. **Always compute from current date** — never hardcode (off-by-year bugs happen; the right value for 2026-05-05 is `1778025600`, NOT `1746403200`).

   ```python
   import time
   oldest = int(time.time() - 14*86400)
   ```

2. **Global Slack search** for `<account_name>` (last 14d). The Slack search tool returns mixed channel + DM hits. Pull the top 50 results.

3. **Search for AE's DMs and group DMs** — `from:@<ae_first_name> <account_name>`. Group DMs surface negotiations.

4. **Search for the AE's own 1:1 DMs** — search `from:@<ae> <account_name>` AND also pull the AE's 1:1 with the user invoking this skill if relevant. These contain the candid posture commentary that doesn't make it to channels.

5. **Iterative external-contact discovery (CRITICAL — do not skip):**
   - From the first-pass Slack and Gong results, extract every external person name that appears (e.g. "Brent", "Ajay", "Nassim", "Karen").
   - For each name, run `slack_search_users` with just the first name. Slack profile/email-domain filters are unreliable; first-name search is the most consistent way to find connected external Slack users.
   - For each matched user, search for messages and group DMs they participate in within the last 14 days. This is how you find threads like Brent Newton's group DM about the Madrid sessions.

6. **Look at all account-related channels** via `slack_search_channels` patterns:
   - `#ext-cursor-<account>`, `#ext-<account>-cursor`
   - `#internal-<account>-*`
   - `#<account>-*`
   - The cross-cutting `#team-field-eng` (often used to recruit FE support for account onsites — was where the Madrid signal first appeared)

7. **Discover keywords iteratively from the first pass.** Topical keywords like `Madrid`, `EBR`, `renewal`, `onsite`, `Q&A` only emerge after the first search. Re-search Slack with these to find adjacent context the account-name-only query missed.

### 5. Gather Notion meeting notes signal — Granola transcripts are how off-Gong calls become discoverable

**Critical:** customer-hosted calls (off-Gong) almost always have a Notion meeting note captured by Granola or similar. The note's title is generic (`Meeting ‣` / `Meeting @Today`), but the **parent page** is named `"<your name> and <contact name>"` (e.g. `"Pranathi Tupakula and Brent Newton"`). All meetings between two people get grouped under that parent page.

This means once you've discovered an external contact name in step 4 (e.g. Brent Newton from the Slack DMs), you can find every Notion-captured call with them in one shot by looking for the parent-page pattern.

Workflow:

1. Run `notion-query-meeting-notes` to get all recent meetings (returns generic-titled list).
2. For each meeting created in the last 14d, look at the `ancestor-path` — specifically the immediate `parent-page` title. If it matches `"<user> and <external contact>"` for any external contact discovered in step 4, that meeting is a captured off-Gong call.
3. Fetch the meeting page with `include_transcript=true` to get the full transcript + action items.
4. Use the **action-items section** of the Notion note directly as input for the "Next steps" section of the Account Plan note — these are already in the right format.

Cap fetches at 10 per account to avoid blow-up.

**Anti-pattern:** do not try to match Notion meetings by title alone — they're all "Meeting ‣". Use the parent-page name (which encodes who attended).

### 6. Synthesize

You now have, for this account:
- AE name + email + role
- Open opportunities (renewal stakes, dollar amount, close date, next step)
- N Gong calls with briefs + key points + verbatim utterances
- M Slack messages (channels + DMs)
- K Notion meeting notes

Produce two artifacts:

**(a) Summary one-liner** — replaces the existing `Summary` cell. Match the existing terse tone (5-25 words, one sentence). Lead with the most material thing happening:
- If there's an active renewal → renewal stakes first
- If there's an upcoming onsite/EBR → name it + date
- If there's a blocker the AE is working through → that
- Otherwise → state of adoption / current focus

Examples calibrated against existing values:
- "Cloud agents use is popping off"
- "Angry about MAX mode"
- "Ran an EBR and Roadmap session 3/12. Need to onboard to private workers."

**(b) Account Plan page** — a new page in the `Account Plans` data source. Title: `<Account> — Account Refresh <YYYY-MM-DD>`. Status: `In progress`. Set the `Client` relation to the Book of Business row's URL.

Page body template:

```markdown
# <Account> — Account Refresh <YYYY-MM-DD>

**AE:** <Name> (<email>) — <role>
**Open opp:** <if any: Name — $X • Stage • Close date • Description>
**Coordination channels:** <list of relevant Slack channels with IDs>

## Last 14 days at a glance
<chronological table: date | event | source>

## Renewal posture
<only if there's an open renewal opp; covers commercial terms, blockers, asks>

## <Major theme 1>
<bullets — could be an upcoming onsite, a key technical blocker, an enablement track>

## <Major theme 2>
...

## Key contacts at <Account>
<bullets of people surfaced from sources; include their role and where they showed up>

## Open threads / blockers
<bullets>

## Next steps
- [ ] **<owner>** — <action>
...

## Verbatim quotes
<2-5 short verbatim quotes from Gong utterances that anchor key points>

## Sources
- Gong: <links to each call>
- Slack: <links to each relevant channel + key thread permalinks>
- Salesforce: <opp name + key fields>
- Notion meeting notes: <if any matched>
```

### 7. Write back to Notion

Use `notion-update-page` to set the Book of Business row's `Summary` to the one-liner and `Up to date` to `__YES__` if any Gong call OR Notion meeting was found in the last 14 days, else `__NO__`.

Use `notion-create-pages` to create the Account Plan page in the Account Plans data source. Set the `Client` relation to point back at the Book of Business row.

### 8. Skip empty accounts

If an account has **zero** signal in the last 14 days (no Gong, no Slack messages mentioning it, no Notion meeting notes), do NOT overwrite the existing Summary or create an Account Plan note. Set `Up to date` to `__NO__` and move on. Log a one-line note that this account had no signal.

## Status logic

`Up to date` = checked (`__YES__`) when **any of**:
- ≥1 Gong call in the last 14 days matched this account, OR
- ≥1 Notion meeting note matched this account by title, OR
- ≥1 off-Gong call attested in Slack (e.g. *"call with Brent on their zoom"*)

Slack chatter alone (messages without an attested call) does NOT flip the checkbox — chat is signal but not "a call". The off-Gong rule is essential because customers who host on their own conferencing software would otherwise look stale even when actively engaged.

## Output format calibration (locked)

These were validated on Benchling + Elastic dry-runs. Don't deviate without explicit user feedback:

- **Summary**: 1 terse sentence, 5-25 words. Match existing tone in the view.
- **Account Plan note** sections: AE/opp/channels header, last-14d table, optional Renewal posture, theme sections, key contacts, open threads, next steps, verbatim quotes, sources.
- **Verbatim quotes**: 2-5 short quotes from Gong utterances. Always attribute to speaker + affiliation.
- **Page naming**: `<Account> — Account Refresh <YYYY-MM-DD>` (always new dated page; do not edit existing pages).

## Common bugs to avoid

1. **Narrow Gong filter**: never use only `email_domain` — also include title-match and AE-participation.
2. **Slack oldest timestamp**: compute `now - 14*86400` at runtime. Never reuse a hardcoded value (off-by-year bugs happen).
3. **ADM is not the AE**: do not use the Notion `ADM` person field as the AE. Always look up via Salesforce.
4. **Multiple accounts in Salesforce**: pick the one with an assigned AE, not the pool.
5. **In-person calls**: their speaker_map has conference-room names; the title-match query catches them.
6. **Off-Gong calls**: customer-hosted Zoom/Meet calls are NEVER in Gong. The Slack-mention scan in step 3a is the only way to find them. Skipping this systematically under-counts touches with customers who host their own calls.
7. **External contacts hide in group DMs**: do not assume external contacts are only in `#ext-*` channels. The Madrid Brent Newton thread was a 3-person group DM that no channel-only search would find. Always discover external contact names from first-pass results and search for them by first name.
8. **Slack search AND-matching across user profile + message text is unreliable**: searching `"brent elastic"` returned no users even though Brent Newton is in the workspace. Search first names alone, then enumerate their conversations.
9. **Empty Internal Slack URL**: many accounts have null `Internal Slack` — don't error; just skip the channel read and rely on global search.
10. **Pagination**: Notion meeting-notes and Slack searches paginate. For 14-day windows, the first page is usually enough; if `has_more=true` on Notion meetings, fetch one more page.
11. **Account name variants**: "Kraken Crypto" → company is `payward.com`. "EToro" → `etoro.com`. When in doubt, do a quick Salesforce account lookup to get the canonical company name and try common TLDs.
12. **Do not declare "no signal" until off-Gong scan is done**: a customer with zero Gong calls but active DM negotiations (e.g. through their events lead like Brent Newton) is highly active. Marking them "no signal" because Gong was empty would be wrong.

## Confirmation before writing

When run interactively (not via cron), show the user the proposed Summary + first 10 lines of the Account Plan note body and ask for go-ahead before writing. When run via cron, write without asking.

## Cron wrapping (future automation)

Once this skill is stable, wrap it in a cron automation:
- Trigger: `cron: "0 16 * * 1"` (Mondays 9am PT = 16:00 UTC)
- Tools: `mcp` for Notion + Slack + Databricks
- Prompt: "Run the account-summary-refresh skill on all accounts in the Book of Business view. Write back without asking for confirmation."
