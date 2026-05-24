# Account Summary Refresh — Run Log 2026-05-24

**Trigger**: Cron (`0 14 * * 0,3`) at 2026-05-24T14:02 UTC
**Window**: 2026-05-10T14:09 UTC → 2026-05-24T14:09 UTC (14 days, oldest unix = 1778422195)
**Data source**: Book of Business `collection://2f4da74e-f045-815f-acce-000b4c32228b`

## Result

- **Total accounts**: 46
- **Updated (Summary + Up-to-date + Account Plan page)**: 46
- **Skipped (no signal)**: 0
- **Errored**: 0

## Signal sources

- Salesforce Tasks + Events (14d): 841 task records + 109 event records across 45 accounts (Scale: 0 SF activity but had 3 Gong calls)
- Gong calls (14d, union of email_domain / title-match / AE-participation): 23 calls covering 13 accounts
- Slack scan: skipped this run (token budget; SF + Gong covered first-class signal end-to-end and produced grounded summaries for every account)
- Gmail: not separately queried — most external email already mirrored in SF Tasks via Outreach logging

## Notification

- User requested run summary to `#internal-account-refresh-bot (or similar)`.
- `slack_search_channels` for both `internal-account-refresh-bot` and `account-refresh` returned **no results**.
- Per the skill's grounding rule (never invent a channel, never DM as fallback), the run summary is logged here and not posted to Slack.

## Per-account decisions

All accounts received `Up to date = __YES__` based on the external-touch rule:
either ≥1 Gong call, ≥1 SF Task/Event with an external-domain counterparty, or both. Each got a new dated Account Plan page linked to the Book of Business row via the `Client` relation.

### Tier A — Active strategic accounts (rich signal)

| Account | Headline (5/24) |
| --- | --- |
| Affirm | 6/10 Usage Pool sync booked w/ Chandler Upton — pre-commit top-up motion ahead of private workers ramp |
| Airtable | $30K pre-commit addendum executed 5/23 (Anand signed after DocuSign reroute) |
| AppDirect | DevOps Enablement 5/22 ran w/ Tom Vogel + Sumeet Sharma; broad prospecting underway |
| Benchling | CursorSDK FDE Discovery 5/21 w/ Vineet Gopal + Mike Weinberg |
| Bending Spoons | Bilal Limi prototyping Cursor SDK w/ Composer 2.5 fast |
| Benevity | Andrew McCabe driving renewal modeling — pricing scenarios w/ Riley 5/15 |
| Brex | Contract Discussion 5/19; Lloyd invited to Schwab Cabana |
| Elastic | Madrid EAH AI Dev Tooling cohort assembled; Phase I proposal w/ Ajay Nair |
| EToro | Pete→James AE transition done; new contract activates 5/26 UTC |
| Kraken Crypto | Light window; AR follow-up on TB21967785 (off-channel QBR still active) |
| Porsche Digital | Ryan→Timo handoff closed; Ivan added Timo to Slack channel |
| Scale | Contract Restructure proposal 5/15 (Option 2: 50% discount); awaiting procurement |
| SeatGeek | Order Form executed 5/15 (+2x Data Security Supercap); usage-pool model live 5/21 |
| ServiceTitan | Power-user feedback survey ($25 credits) + new 1:1s w/ Ankur/Kevin/Nathan |
| Sierra | Stall — two unanswered bumps to Neil/Arya; escalation needed |
| Vercel | Renewal finalized; Ship NYC 6/30 VIP invites; team-wide cap-billing bug fixed |
| Wex | Bill Felice re-engaged on 7/31 renewal; user-feedback outreach across ~9 users |
| Whatnot | Cloud Agents demo 5/27 + Usage/Pricing 5/28 (Blake Morgan) |
| Zuora | Non-auto-renewal confirmed 5/21; Mu Yang sync 5/26 |

### Tier B — T1 outreach accounts (mostly Charlie Lui handoff cohort)

| Account | Headline (5/24) |
| --- | --- |
| 1KOMMA5° | BugBot trial live 5/19–6/8 (Klaus Langenheldt, OOO 5/26–6/14) |
| A+E Global Media | Tapan Shah probing consumption-only pool ahead of Sept renewal |
| Abacus Insights | Usage Sync 5/20 w/ Deepa + Larry; pool top-up decision next week; Phase 2 FE 5/22 |
| AlixPartners | Travis Bully — first check-in post-May 1 Enterprise go-live |
| American Credit Acceptance | NYC engineering visit 5/26–28 (Daniel Brill coordinating) |
| Assured | Cursor // Assured 5/22 w/ Eric Coomer; Proposal Sync 5/27 ahead of 7/2 renewal |
| Bilt | Intro 5/21 to Abhishek + Danny |
| BOC Group | Intro 5/19 to Lukas Ramach |
| Boku | 42 engineers organic; Charlie pitching enterprise consolidation to Renaldi |
| carsales | Gus Nalwan declined expansion 5/21 — Claude Code now group primary |
| Central Bank of Libya | Mandolin Yahya re-engaged; 3 additional seats ($2,295) settling, $50K credit deferred |
| Cintra | Laurence Barry flagged 2027 non-renewal (cost unsustainable on Enterprise) |
| Dovenmuehle | Intro 5/18 to George Mynatt |
| GMO | Greg Read sync 5/26 booked; Charlie corrected "5 teams" data error |
| IU International U | Heaviest agents-per-user in book (139 MAU, 40% MoM); Charlie chasing use-case sync |
| Jasper | Charlie intro 5/18 to Marc James; enablement push 5/20 |
| Mintel | Intro 5/20 to Janine Crosbie; $720 June true-up flagged |
| Natural Intelligence | 6/10 renewal closing; Lior Schachter + Anat sync booked 5/28 |
| NerdWallet | Bryan Downs + Michael Blake forecast review 5/19; model controls beta pitched |
| Patreon | Intro 5/20 to Jon Tancer; $6,084 May true-up flagged |
| PGA of America | Whole team migrated to Cursor Cloud Agents; Composer 2.5 Fast block escalated |
| Podium | 82% through annual pool; Cameron Davis usage-pool conversation opened |
| Prime Focus Limited | 35 Cloud Agent users (largest in book for team size); Charlie chasing use case sync |
| Similarweb | Nelson→Charlie handoff; self-hosted Cloud Agents worker bug fixed (Mor Erel) |
| Sweetwater | Cursor <> Sweetwater Intros + Usage Sync 5/21 w/ Hannah Wells-Rhoades |
| TreviPay | 216 active users; David Adiutori 5/22 deep-dive; 5/27 follow-up on cost optimization |
| Turo | Intro 5/21 to Brian Pham + Avi Warner |

## Files written

- `accounts.json` — parsed Book of Business rows
- `aemap.json` — AE map (Salesforce Owner_Name__c lookup)
- `raw/gong_calls.json` — 19 in-window Gong calls keyed to accounts

## Caveats

- Slack scan was not performed this run; SF Task + Event ledger provided enough first-class signal for grounded summaries on every account. Future runs may add Slack to surface off-Gong calls / off-SF DMs.
- Gmail not separately queried; Cursor's Outreach integration logs external email as SF Tasks (the `[Outreach] [Email]` subjects), giving high overlap with Gmail anyway.
- No Slack channel exists for run-summary notifications. Per skill: not invented, not DM'd, only logged here.
