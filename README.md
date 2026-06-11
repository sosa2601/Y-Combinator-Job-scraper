# YC Company Scraper & GTM Scorer
### VysionAI · n8n Self-Hosted · Saswati Gorai

A weekly workflow that pulls all Y Combinator companies, filters them against ICP criteria, scores each one for GTM fit, generates a personalised AI outreach hook, and writes the results to a Google Sheet tracker.

---

## System Overview

```
YC Public API → ICP Filter → Deduplicate → Batch of 10
  → GTM Gap Score → Build Hook Prompt → Claude (OpenRouter)
  → Attach Hook → Append to Master Tracker Sheet
```

**Schedule:** Every Sunday at 6am
**Output:** One row per new qualifying company in the `Master Tracker` sheet tab

---

## Node-by-Node Breakdown

| # | Node | Purpose |
|---|---|---|
| 1 | Weekly Schedule — Sunday 6am | Triggers the workflow once a week |
| 2 | Fetch All YC Companies | GET request to the YC public companies API — returns the full list |
| 3 | Filter by ICP Criteria | Code node — filters by tag match and team size (see ICP logic below) |
| 4 | Read Existing Slugs from Sheet | Reads all `YC Slug` values already in the Master Tracker (runs once) |
| 5 | Remove Already Scraped Companies | Deduplicates — drops any company whose slug is already in the sheet |
| 6 | Process in Batches of 10 | Splits remaining companies into batches of 10 to avoid rate limits |
| 7 | Compute GTM Gap Score | Scores each company 0–100 based on GTM fit signals (see scoring below) |
| 8 | Build Hook Prompt | Builds the Claude prompt per company using name, one-liner, tags, team size, score |
| 9 | Hook Generator | POST to OpenRouter → Claude Haiku — generates a 1-sentence cold outreach opener |
| 10 | Attach AI Hook to Company Row | Extracts the hook text and merges it into the final row object |
| 11 | Append Row to Master Tracker | Appends the complete row to the Google Sheet |
| 12 | Log Completion | Logs `done` status once all batches finish |

---

## ICP Filter Logic

Applied in the `Filter by ICP Criteria` node. A company must pass **all three** conditions:

**1. Status** — not `Inactive` or `Dead`

**2. Team size** — between 10 and 150 employees

**3. Tag match** — at least one tag from this set:
```
b2b, saas, sales, marketing, analytics, crm, enterprise,
ai, automation, fintech, hr tech, developer tools, revenue
```

Companies that don't meet all three are dropped before scoring.

---

## GTM Gap Scoring Model

Each company is scored out of 100. Points are additive:

| Signal | Points | Logic |
|---|---|---|
| Team size 15–80 | +15 | Sweet spot for a lean GTM hire |
| Not actively hiring | +20 | GTM gap signal — growth without dedicated GTM headcount |
| Has sales / CRM / revenue tag | +20 | Direct GTM relevance |
| Both B2B + SaaS tags | +15 | Core ICP match |
| Automation or analytics tag | +10 | Tool affinity |
| Fintech / HR / Enterprise industry | +10 | High-value verticals |

**Score interpretation:**

| Score | ICP Fit | Priority |
|---|---|---|
| 60–100 | High | 1 |
| 35–59 | Medium | 2 |
| 0–34 | Low | 3 |

Maximum score is capped at 100.

---

## AI Hook Generation

For each company, Claude generates a single cold email opener sentence via OpenRouter.

**System prompt given to Claude:**
> "You write ultra-short cold email openers (1 sentence max). You are Saswati, a GTM Engineer. Be specific, human, no buzzwords."

**User prompt includes:**
- Company name
- One-liner (what they do)
- Tags
- Team size
- GTM Gap Score

**Model:** `anthropic/claude-haiku-4.5` · Max tokens: 120
**Output:** Written to the `AI Hook` column in the sheet

---

## Master Tracker Sheet Columns

The workflow writes the following columns per company:

| Column | Source |
|---|---|
| Company Name | YC API |
| YC Slug | YC API (used for deduplication) |
| Batch | YC API (e.g. W23, S24) |
| Status | YC API |
| Team Size | YC API |
| Industry | YC API |
| Sub-industry | YC API |
| Tags | YC API |
| Is Hiring | YC API |
| Stage | YC API |
| Top Company | YC API |
| Website | YC API |
| YC Profile URL | YC API / constructed |
| LinkedIn URL | YC API |
| Crunchbase URL | YC API |
| Regions | YC API |
| HubSpot? | Blank — fill manually or via enrichment |
| Intercom? | Blank |
| Segment? | Blank |
| Mixpanel? | Blank |
| GA4? | Blank |
| Drift? | Blank |
| Apollo? | Blank |
| SalesLoft? | Blank |
| Open Roles | Blank — fill manually |
| GTM Roles | Blank |
| Latest GTM Role | Blank |
| Days Since Post | Blank |
| GTM Gap Score | Computed (0–100) |
| ICP Fit | Computed (High / Medium / Low) |
| Priority | Computed (1 / 2 / 3) |
| One-Liner | YC API |
| AI Hook | Claude-generated outreach opener |
| Outreach Status | Default: `Not Started` |
| Contact LinkedIn | Blank — fill manually |
| Notes | Blank |

---

## Configuration

### To change ICP tag filters
In `Filter by ICP Criteria`, edit the `TARGET_TAGS` set:
```js
const TARGET_TAGS = new Set(['b2b','saas','sales', ...]);
```

### To change team size range
In the same node, update:
```js
if (size < 10 || size > 150) return false;
```

### To change scoring weights
In `Compute GTM Gap Score`, adjust the `+N` values on any scoring condition.

### To change the AI model
In `Hook Generator`, update the `model` field in the JSON body:
```json
"model": "anthropic/claude-haiku-4.5"
```

### To change the outreach persona in the hook prompt
In `Build Hook Prompt`, edit the system prompt string in the code node.

### To change the run schedule
Update `Weekly Schedule — Sunday 6am` — currently set to weekly on Sunday at 6am.

---

## Dependencies

| Credential | Node | Type |
|---|---|---|
| Google Sheets OAuth2 | Read Existing Slugs, Append Row | OAuth2 |
| OpenRouter API key | Hook Generator | Bearer token in Authorization header |

---

## First Run Checklist

- [ ] Create the Google Sheet with a `Master Tracker` tab
- [ ] Add column headers matching the columns listed above (row 1)
- [ ] Connect Google Sheets OAuth2 credential in n8n
- [ ] Add your OpenRouter API key to the `Hook Generator` HTTP node Authorization header
- [ ] Import the workflow JSON into n8n
- [ ] Run manually once to verify ICP filtering and scoring output
- [ ] Check that AI hooks are being generated and written to the sheet
- [ ] Activate the workflow for weekly scheduled runs

---

## Troubleshooting

**No companies returned after filter**
- YC API may have changed structure — check the raw response from `Fetch All YC Companies`
- Tag values are case-sensitive in the API — verify tag names match the `TARGET_TAGS` set

**Duplicate companies appearing**
- Check the `YC Slug` column exists in the sheet and is populated — deduplication reads from this column
- If the column is empty on existing rows, the dedup check will not work

**Hook Generator returning empty**
- Verify the OpenRouter API key is valid and has credits
- Check the `_hook_body` field is being constructed correctly in `Build Hook Prompt`

**Workflow runs but sheet has no new rows**
- All fetched companies may already exist in the sheet (deduplication working correctly)
- Reduce ICP filter strictness temporarily to test

---

*Built by Saswati Gorai · VysionAI · vysionai.com*
