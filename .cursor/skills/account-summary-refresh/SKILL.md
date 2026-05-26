---
name: account-summary-refresh
description: Refresh the Summary and Up-to-date status on the Book of Business Notion view by synthesizing the last 14 days of Gong calls, Notion meeting notes, Slack messages (channels + DMs), external email (via gws/Gmail), and Salesforce activity for each account. Writes a dated Account Plan note with the deeper bullets. Use when the user says "refresh accounts", "update book of business", "refresh account summaries", "weekly account refresh", or wants any account row in the Book of Business resynced from recent signal.
---

# Account Summary Refresh

Refreshes the **Book of Business** Notion view (`https://www.notion.so/cursorai/2f4da74ef045810eb479f860344b6f3b`) by pulling the last 14 days of customer signal for each row and writing back a 1-line Summary + a dated Account Plan note. Designed to be invokable ad hoc and also wrapped in a weekly cron automation.

## Audience and lens (READ THIS FIRST)

The reader of these summaries and account plans is an **AI Deployment Manager (ADM)**. Their job is to drive **broad and deep adoption of Cursor** inside the account. Frame every output around adoption, not around tactical support.

What counts as material signal for this lens:

- **Breadth**: new teams/orgs/seats adopting Cursor, expansion to new BUs, rollout milestones.
- **Depth**: usage of advanced surfaces (Cloud Agents, MAX mode, Bugbot, Background Composer, custom MCPs, enterprise features), power-user growth, model mix changes, agent adoption.
- **Strategic motions**: EBRs, onsites, executive sponsorship, renewal posture, commercial expansion, multi-year deals, displacement of competitors.
- **Adoption blockers that need ADM action**: enablement gaps, training needs, security/legal blockers gating rollout, change-management pushback, exec misalignment.
- **Champions and detractors**: who is advocating internally, who is pushing back, who needs enabling.

What is **explicitly out of scope** (filter these out — do not surface in Summary or Account Plan):

- Individual bug reports, error messages, crash reports, ticket-style support questions in Slack.
- One-off product complaints that do not affect rollout (e.g. "X feature is slow today", "Y stopped working in my repo").
- Engineering-level back-and-forth about specific repros, line numbers, model behavior on a single prompt.
- Billing/admin questions that are not commercial strategy.

A bug report is only relevant if it has become an **adoption blocker** (e.g. "we paused the Cloud Agents rollout because of repeated timeouts" → in scope; "got a 500 in cmd-K this morning" → out of scope). When in doubt, ask: *does this change what the ADM should do this week to grow adoption?* If no, drop it.

## Grounding rule (READ THIS SECOND)

**Every claim in the Summary or Account Plan must be grounded in a primary source.** Do not infer, guess, or fill in plausible details. If a fact isn't stated explicitly in the data you gathered, leave it out.

Specifically forbidden inferences:

- **Location from area codes** — `+1-312-…` does NOT mean Chicago. It means the phone number was issued in the 312 area. People move and keep numbers.
- **Location from time zones** — `CDT` does NOT mean Chicago. Central time covers a large chunk of the US and the invite could have been scheduled in CT for any reason.
- **Location from snippet hints** — "see you at the hotel" implies someone traveled, not where. Don't name a city unless it's explicitly written.
- **Identity from first names** — "Brent said…" without an email or Slack profile attached is ambiguous. Tie names to a domain or profile before using them.
- **Dates from calendar invite subjects** — an invite titled "@ Mon May 4" tells you the *scheduled* time, not whether the meeting actually happened. Confirm via attendance signal (Gong recording, post-event email, calendar acceptance) before citing it as a completed event.
- **Account intent from one quote** — a single Slack message or email line is not the customer's posture. Triangulate across ≥2 sources before claiming "they're unhappy with X" or "they want Y firmwide".
- **Slack channel names that you have not verified** — do not assume `#internal-<x>`, `#account-refresh-bot`, `#ext-<account>`, or any other channel exists. Call `slack_search_channels` and confirm the channel id before referencing it. If it doesn't exist, do not invent a fallback ("DM'ing you instead", "posting to #general") — just do not post.

When you only have indirect signal, **say so explicitly** in the note (e.g. "session was in-person on 5/4; city not stated in sources"). It is always better to be uncertain than fabricated.

Citation discipline: for every line in the "Last 14 days at a glance" table, include the source (Gong call id, Slack permalink, SF activity id, or email thread subject + date). The Summary one-liner doesn't need inline citations, but everything it says must trace to a row in that table.

## Required MCP servers (must be authenticated)

- `dashboard-team-1-Notion` — read view + write Summary/checkbox + create Account Plan pages
- `dashboard-team-1-Databricks SQL` — Gong calls + Salesforce mirror
- `dashboard-team-1-Slack` — channel reads, search, DMs

If any of these only expose `mcp_auth`, call the auth tool and ask the user to accept the prompt before continuing.

## Required CLI tools

- `gws` (Google Workspace CLI, `/opt/homebrew/bin/gws`) — Gmail signal gather. Authenticated as `pranathi@anysphere.co`. Prefer this over the Gmail MCP. See workspace rule `gmail-drafts-via-gws.mdc`.

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
    user:        revops.pt_salesforce.user
slack:
  internal_workspace_domain: anysphere.slack.com
  enterprise_workspace_domain: anysphere.enterprise.slack.com
owner:
  # The ADM these refreshes are written for. Used to filter the Book of Business
  # data source so we only touch accounts where Pranathi is the ADM (Notion) or
  # the TAM (Salesforce) — see step 1.
  name: Pranathi Tupakula
  email: pranathi@anysphere.co
  notion_user_id: 2e5d872b-594c-81f6-a518-00029c98377d
  salesforce_user_id: 005Hr00000ImOPnIAN
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

## Outputs and write boundary (STRICT)

This skill writes to exactly two destinations. Anything else is out of scope.

**Allowed writes:**

1. Update the `Summary` text and `Up to date` checkbox on the Book of Business row for each account in scope (via `notion-update-page`).
2. Create a new page in the Account Plans data source titled `<Account> — Account Refresh <YYYY-MM-DD>` with the `Client` relation pointing back at the Book of Business row (via `notion-create-pages`).

**Forbidden — do NOT do any of these, ever:**

- Post any Slack message, in any channel, to any user, in any thread. The Slack MCP is for **read** only in this skill.
- Send any email, draft any email, or open any compose window. The `gws` CLI is for `gmail users messages list` / `messages get` only — never `messages send`, `drafts create`, or similar.
- DM the user with status, progress, errors, or fallback messages. If something fails, write a single line to stdout/log and continue or exit; do not "helpfully" reach out via DM.
- Invent or assume the existence of a notification channel (`#internal-account-refresh-bot`, `#adm-status`, etc.). These do not exist. Do not propose creating one.
- Write to any Notion page, database, or data source other than the two listed above.
- Modify Salesforce, Databricks, or any other system. All non-Notion sources are read-only.

If a future variant of this skill needs to notify a human, the destination must be passed in as an explicit input parameter (e.g. `--notify-channel <id>`), not inferred. Until that parameter exists in `## Inputs`, no notification.

## Workflow

### 1. Pull the Book of Business rows (filtered to the ADM/TAM)

**Scope rule (READ FIRST):** the underlying Book of Business data source occasionally has rows bulk-imported by a Salesforce-sync pipeline that are not Pranathi's accounts (most often Charlie Lui's T1 outreach book, every row with `Band = "T1"` and `"Account ID"` populated). These rows must NOT be refreshed. The discriminator is membership in the ADM/TAM set:

- **Notion `ADM`** must contain Pranathi's `notion_user_id` (`2e5d872b-594c-81f6-a518-00029c98377d`), **OR**
- the row's Notion `Account ID` must match a Salesforce account whose `Technical_Account_Manager__c = '005Hr00000ImOPnIAN'` (Pranathi's `salesforce_user_id`).

Most of the time the first condition alone is enough; the SF TAM check is a safety net for any new row that gets added to Notion without an ADM person set.

Step-by-step:

1. **Pull SF Account IDs where Pranathi is TAM** (drives the safety-net filter):

```sql
-- via execute_sql_read_only on Databricks SQL
SELECT Id, Name
FROM revops.pt_salesforce.account
WHERE Technical_Account_Manager__c = '005Hr00000ImOPnIAN'
```

2. **Pull Notion rows where ADM = Pranathi OR Account ID is in the TAM set:**

```sql
-- via notion-query-data-sources, SQL mode
SELECT Name, Band, "Up to date", Summary, "Internal Slack", "Account ID", ADM, url
FROM "collection://2f4da74e-f045-815f-acce-000b4c32228b"
WHERE ADM LIKE '%2e5d872b-594c-81f6-a518-00029c98377d%'
   OR "Account ID" IN ( <comma-separated quoted SF Account IDs from step 1> )
ORDER BY Band, Name
```

If the SF TAM list is empty (or Databricks is unavailable), fall back to the `ADM` clause alone. **Never** query the data source without a scope filter — that re-includes auto-imported accounts and overwrites work for the wrong AE.

3. **Sanity check:** as of 2026-05-26 this filter returns exactly 18 rows (Affirm, Airtable, AppDirect, Benchling, Bending Spoons, Benevity, Brex, Elastic, EToro, Kraken Crypto, Porsche Digital, Scale, SeatGeek, ServiceTitan, Sierra, Vercel, Wex, Whatnot, Zuora). If you get materially more rows than that, inspect the deltas before writing — an auto-import probably leaked back in.

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

And pull AE-logged activities for the window — **this is a first-class signal source**, not optional. SF activities are where AEs log emails, calls, in-person meetings, and onsites. For accounts where SF is well-maintained, this single query covers most of what Gmail would otherwise give us.

```sql
-- Tasks: emails, calls, to-dos
SELECT Id, Subject, Description, ActivityDate, CreatedDate, OwnerId, Type, Status
FROM revops.pt_salesforce.task
WHERE AccountId = '<account_id>'
  AND ActivityDate >= date_sub(current_date(), 14)
ORDER BY ActivityDate DESC LIMIT 30
```

```sql
-- Events: meetings, calls, onsites (with location field)
SELECT Id, Subject, Description, ActivityDate, StartDateTime, EndDateTime, Location, OwnerId, Type
FROM revops.pt_salesforce.event
WHERE AccountId = '<account_id>'
  AND ActivityDate >= date_sub(current_date(), 14)
ORDER BY StartDateTime DESC LIMIT 30
```

Note: SF table names (`task`, `event`) may vary by mirror — verify against the `revops.pt_salesforce.*` schema before running. The `Location` field on Event is authoritative for in-person meetings — use it directly (do not infer location from phone area codes or time zones; see the grounding rule).

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

8. **Filter Slack hits through the ADM adoption lens** before they reach synthesis. Drop the following classes of messages from the working set — they are noise for an AI Deployment Manager:

   - Bug reports / error reports — patterns like `error`, `500`, `crashed`, `repro`, `stack trace`, `wasn't working`, `broken`, `regressed`, `escalating to eng`, links to Linear/Jira tickets, `cc @<eng-name>` for triage.
   - One-off feature complaints with no rollout impact (e.g. "X is slow today", "Y model gave a bad answer on this prompt").
   - Engineering back-and-forth on a single repro (line numbers, file paths, debug logs).
   - Generic support / billing / admin questions ("how do I reset SSO", "invoice question").

   **Promote** a message back into the working set only if it has clear ADM relevance:
   - It blocks or paces a rollout ("we paused the Cloud Agents pilot because…", "legal is holding up the expansion until…").
   - It is exec-level posture ("the CTO wants a roadmap session", "Brent escalated to their CIO").
   - It signals breadth/depth movement ("the platform team is now onboarded", "they want to turn on MAX mode firmwide").
   - It is a champion/detractor signal ("Karen is pushing this internally", "Ajay is unhappy with Bugbot noise and is pulling support").

   Keep a count of dropped-as-noise messages per account; if it's high (>20) note it as a separate line in the Account Plan's "Open threads / blockers" section like *"High volume of bug-report chatter (~30 msgs) — not surfaced here; flag to product if patterns emerge."* This way the ADM knows volume exists without it polluting the strategic summary.

### 4c. Gather Gmail signal — first-class touch source

External email is a first-class signal alongside Gong, SF activities, and Slack. It uniquely catches:

- Threads where a **non-AE Cursor person** is corresponding (SE, exec sponsor, the ADM themselves, support). These often don't get logged to SF because the AE isn't on the thread to log them.
- **Commercial / legal / security coordination** — order forms, MSAs, security questionnaires, redlines. These are routinely missed in SF.
- **Async-only customers** — accounts deep in procurement or security review may have zero calls in a window but live email threads.
- **Exec-to-exec outreach** that bypasses the AE.

Use the `gws` CLI (workspace rule `gmail-drafts-via-gws.mdc`). The CLI authenticates via `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` — for local runs that defaults to the user's keyring; for cloud-agent runs see the "Cloud agent deployment" section.

```bash
# Derive candidate domains from the account (see step 3b for alias handling).
DOMAINS="<acct>.com OR <acct>.co OR <acct>.io OR <known_alt_domain>"

gws gmail users messages list \
  --params "$(jq -nc --arg q "(from:($DOMAINS) OR to:($DOMAINS) OR cc:($DOMAINS)) newer_than:14d" \
              '{userId:"me", q:$q, maxResults:50}')"

# For each message id:
gws gmail users messages get \
  --params "$(jq -nc --arg id "<message_id>" \
              '{userId:"me", id:$id, format:"metadata", metadataHeaders:["From","To","Cc","Subject","Date"]}')"
```

For each hit, capture: `Date`, `From`, `To`, `Cc`, `Subject`, `threadId`, and the API-provided `snippet`. Group by `threadId` so an 8-message negotiation counts as one thread.

**ADM lens filter** (same as Slack — see step 4.8). Drop:
- GitHub / Linear / Jira / monitoring / status-page notifications.
- Auto-generated mail (calendar invites that are pure ICS, bounce notifications, OOO replies).
- Internal-only threads where the account name appears but no `@<account>` address is actually in From/To/Cc.
- Individual bug reports / support tickets.

Promote:
- Commercial / legal / security threads (subjects like "Order Form", "MSA", "DPA", "Security Questionnaire", "Redlines", "Pricing").
- Exec-to-exec ("CIO", "CTO", "VP", "Chief").
- Rollout coordination ("onboarding", "rollout", "expansion", "enablement", "kickoff", "EBR", "QBR").
- Post-meeting follow-ups from named external contacts.

**Dedup against SF activities (CRITICAL).** If an AE diligently logs emails as `Task` records in SF (step 2), the same email will appear in both sources. Deduplicate before synthesis:
- Match by `Subject` (normalize whitespace, strip `Re:`/`Fwd:` prefixes) + `Date` within ±1 day, OR
- Match by SF Task `Description` containing the Gmail Subject text.
Treat dedup'd touches as a single row with both source cites attached, not as two separate events.

Cap at ~30 messages per account after dedup; if more, sample most-recent-per-thread.

**Citation format.** In the Account Plan note's "Last 14 days at a glance" row, cite Gmail touches as:
`Email · "<Subject>" · <Date> · <From-name> (<from-domain>)`
Do **not** paste snippets verbatim except as deliberately-selected verbatim quotes with attribution. Do not infer facts from snippets (see Grounding rule). If the snippet implies something the headers don't confirm (e.g. a location, an outcome), do not surface that implication.

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
- SF activities (Task + Event records) in the window — AE-logged touches
- N Gong calls with briefs + key points + verbatim utterances
- M Slack messages (channels + DMs), filtered through the ADM lens
- E external email threads (Gmail), filtered through the ADM lens, deduped against SF
- K Notion meeting notes

Produce two artifacts:

**(a) Summary one-liner** — replaces the existing `Summary` cell. Match the existing terse tone (5-25 words, one sentence). Lead with the most material **adoption** thing happening — what should the ADM focus on this week to grow breadth or depth?

Priority order for what to lead with:
- Active renewal stakes (commercial gate on continued adoption).
- Upcoming exec moment (EBR / onsite / roadmap session) with date.
- Rollout milestone or blocker (new BU onboarding, paused pilot, security gate).
- Depth-of-use movement (cloud agents popping off, MAX mode adoption, new advanced-feature uptake).
- State of adoption / current ADM focus.

Do **not** lead with bug reports, tactical support issues, or one-off complaints. Those belong (if at all) in the Account Plan's blockers section, not the Summary.

Examples calibrated against existing values:
- "Cloud agents use is popping off"
- "Ran an EBR and Roadmap session 3/12. Need to onboard to private workers."
- "Madrid onsite 5/22 with Brent; goal is firmwide MAX mode rollout."
- "Renewal in 6w; platform team adopted, but security blocking BU expansion."

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
- Email: <thread subjects + dates; do not paste full bodies>
- Salesforce: <opp name + key fields>
- Notion meeting notes: <if any matched>
```

### 7. Write back to Notion

Use `notion-update-page` to set the Book of Business row's `Summary` to the one-liner and `Up to date` to `__YES__` if **any external touch** (call, Slack DM/message from an external user, or external email) was found in the last 14 days, else `__NO__`. See "Status logic" below for the precise rule.

Use `notion-create-pages` to create the Account Plan page in the Account Plans data source. Set the `Client` relation to point back at the Book of Business row.

### 8. Skip empty accounts

If an account has **zero external signal** in the last 14 days (no Gong call, no Slack messages with an external participant mentioning it, no external email, no Notion meeting notes), do NOT overwrite the existing Summary or create an Account Plan note. Set `Up to date` to `__NO__` and move on. Log a one-line note that this account had no signal.

If the account has signal but no call (e.g. live email negotiation, or active external DM thread), DO write a Summary + Account Plan note — frame it around the touch type ("Async negotiation on order form; no calls this window"). Don't pretend a meeting happened.

## Status logic

`Up to date` = checked (`__YES__`) when **any one of**:
- ≥1 Gong call in the last 14 days matched this account, OR
- ≥1 SF Task or Event with `ActivityDate` in the last 14 days, OR
- ≥1 Notion meeting note matched this account (via the parent-page convention from step 5), OR
- ≥1 off-Gong call attested in Slack (e.g. *"call with Brent on their zoom"*), OR
- ≥1 Slack message in the last 14 days **authored by an external user** at this account (DM, group DM, or shared channel), OR
- ≥1 email in the last 14 days to or from an `@<account>` domain after ADM-lens filtering.

Internal-only Slack chatter (Cursor employees talking about the account with no external participant in the thread) does NOT flip the checkbox — that's internal awareness, not an account touch. It can still inform the Summary and Account Plan body, but it's not a "touch".

The external-touch rule is essential because customers in security review, procurement, or async commercial motions often go a full window with zero calls but live email/DM activity, and they would otherwise look stale despite being actively engaged.

## Output format calibration (locked)

These were validated on Benchling + Elastic dry-runs. Don't deviate without explicit user feedback:

- **Summary**: 1 terse sentence, 5-25 words. Match existing tone in the view. Adoption-framed; never a bug report.
- **Account Plan note** sections: AE/opp/channels header, last-14d table, optional Renewal posture, theme sections, key contacts, open threads, next steps, verbatim quotes, sources.
- **Theme sections** are adoption-themed (rollout, depth-of-use, exec engagement, enablement, competitive displacement) — not bug-triage themed.
- **Verbatim quotes**: 2-5 short quotes from Gong utterances. Always attribute to speaker + affiliation. Prefer quotes that reveal posture, intent, or champion/detractor stance — not quotes about specific bugs.
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
13. **Bug reports leaking into the Summary / Account Plan**: this skill is for an AI Deployment Manager focused on broad and deep adoption — not for product support triage. Filter out individual bug reports, error messages, and repro back-and-forth at the Slack-gather step (see step 4.8). Only surface a bug if it has become a rollout-level blocker. The Summary one-liner must never lead with a bug.
14. **Treating internal-only Slack chatter as a touch**: a thread between Cursor employees about an account, with no external participant, does NOT flip `Up to date`. The external-participant check matters — verify the author of at least one message in the matched thread belongs to the customer (non-Anysphere email domain, or guest user from the account's workspace). Internal awareness is informational; only external participation is a touch.
15. **Missing async-only accounts**: an account with zero calls but live commercial coordination (order form, security questionnaire, exec exchange) is highly active. Verify all of Gong + SF Tasks/Events + Slack + Gmail (steps 2, 3b, 4, 4c) were actually queried before declaring `Up to date = __NO__`. SF-only is not enough — non-AE Cursor people corresponding by email is exactly the gap Gmail closes.
16. **Double-counting AE-logged emails**: if an AE logs an outbound email as a SF `Task` AND that same email lives in Gmail, you'll see both. Dedup by normalized Subject + ±1 day before counting touches (see step 4c dedup section). The combined row should carry both source cites.
17. **Inferring facts the sources don't state**: never fabricate location, identity, timing, or intent. Phone area codes do not tell you a city. Time zones do not tell you a city. First names without an attached domain do not identify a person. Calendar invite subjects tell you what was scheduled, not what happened. If a fact isn't in your gathered sources, leave it out or say "not stated in sources". See the Grounding rule at the top of this skill.
18. **Citing a row without a source**: every line in the Account Plan's "Last 14 days at a glance" table must have a primary-source cite (Gong call id, Slack permalink, SF activity id, or email Subject+Date). If you can't cite it, you can't include it.
19. **Inventing Slack channels or DM fallbacks**: a previous run posted "(#internal-account-refresh-bot doesn't exist yet — DM'ing you instead.)". That channel does not and will not exist, and DM'ing the user was never an authorized output. This skill writes ONLY to the two Notion destinations in the "Outputs and write boundary" section — never Slack, never email, never DMs, never a notification channel. Status / progress / errors go to stdout. Full stop.
20. **Refreshing accounts that aren't Pranathi's (ADM/TAM scope leak)**: the Book of Business data source occasionally has rows bulk-imported by a Salesforce-sync pipeline pushing other reps' books into the same collection (most often Charlie Lui's T1 outreach book — every row with `Band = "T1"` and `"Account ID"` populated, no `ADM` person set, names stuffed with `<Account>\nRenewal <date>\nARR $X`, blank page body, and Salesforce-shape `Champion` / `Risk` / `Stage` / `Current ARR` / `Expansion Potential` / `Days Since Touch` auto-fields). The 2026-05-24 run swept ~27 such rows in and wrote Summaries + Account Plan pages on them before they were trashed by Pranathi. Always filter step 1 by `ADM = Pranathi's notion_user_id` OR `"Account ID"` ∈ (SF accounts where `Technical_Account_Manager__c = Pranathi's salesforce_user_id`). The simple test: if a row has `Band = "T1"` and `Account ID` populated and `ADM` empty, it is NOT in scope.

## Confirmation before writing

When run interactively (not via cron), show the user the proposed Summary + first 10 lines of the Account Plan note body and ask for go-ahead before writing. When run via cron, write without asking.

## Cloud agent deployment

Once stable, wrap this skill in a Cursor Cloud Agent on a weekly cron. Key wiring:

### Trigger

```yaml
schedule: "0 16 * * 1"  # Mondays 09:00 PT = 16:00 UTC
prompt: |
  Run the account-summary-refresh skill on all accounts in the Book of
  Business view. Write back without asking for confirmation.
```

### MCP servers (HTTP)

- Notion (read + write Book of Business + Account Plans)
- Slack (read channels/threads/users)
- Databricks SQL (Gong + Salesforce mirror)

Credentials for these stay on the MCP side — they never enter the VM. If you migrate the cron to "Team Owned", every MCP must be re-OAuth'd as the team service account.

### `gws` CLI for Gmail signal (step 4c)

`gws` runs inside the agent VM and needs non-interactive auth. Pick one:

| Path | Credentials file | Acts as |
|---|---|---|
| Exported OAuth (refresh token) | `gws auth export --unmasked > credentials.json` on your laptop | The user who exported (e.g. `pranathi@anysphere.co`) |
| Service Account + Domain-Wide Delegation | SA JSON from GCP with Gmail readonly scope | A workspace user the SA impersonates |

**Install `gws` in the snapshot or Dockerfile**, not in `install` — `install` runs every boot and reinstalling the CLI wastes startup time.

**Wire the credentials** as a redacted environment-scoped secret:

```
Secret name: GWS_CREDENTIALS_JSON
Value:       <contents of credentials.json>
Redacted:    yes
```

In the boot hook, materialize it to a file:

```bash
mkdir -p /home/ubuntu/.config/gws
printf '%s' "$GWS_CREDENTIALS_JSON" > /home/ubuntu/.config/gws/credentials.json
chmod 600 /home/ubuntu/.config/gws/credentials.json
```

Set the persistent env var (non-secret config):

```
GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE=/home/ubuntu/.config/gws/credentials.json
```

### Egress allowlist

If the agent runs on `Default + allowlist`, add Google API hosts so `gws` can reach Gmail:

- `oauth2.googleapis.com`
- `www.googleapis.com`
- `gmail.googleapis.com`

### Scope hygiene

- Use Gmail **readonly** scope on the gws credentials. This skill never sends mail — readonly is enough and limits blast radius.
- Slack MCP: **read-only**. Do not grant any `chat:write`, `chat:write.public`, `im:write`, or DM-send scope. The skill's write boundary (see "Outputs and write boundary") forbids Slack posts, and the scope should mirror that.
- Notion MCP: scope write to the Book of Business view and the Account Plans data source. Do not grant workspace-wide write.

### Smoke test before scheduling

Run the agent interactively once with a sentinel command before turning on the cron:

```bash
gws gmail users messages list --params '{"userId":"me","q":"newer_than:1d","maxResults":1}'
```

If that returns a result, gws auth is wired. If it errors with `invalid_grant` or `credentials not found`, the env var isn't pointed at the file, or the JSON has a stray newline from the secret paste.
