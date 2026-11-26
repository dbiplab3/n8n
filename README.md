# n8n Workflows

A collection of [n8n](https://n8n.io) workflow exports — automations for Jira ticket/taxonomy management, intake forms, and content curation. Each `.json` file is a ready-to-import n8n workflow.

All workflows here have been scrubbed of real credentials, internal IDs, and company-identifying data before being committed — see **Before you import** below.

## Workflows

### [`jira-tickets-v12-reload.json`](jira-tickets-v12-reload.json) — Jira tickets by mapped company
Looks up a company-to-ID mapping table in Postgres, builds batched JQL queries (JQL has a practical length limit, so large company lists are split into chunks), fetches matching Jira tickets, and reloads the results into a Postgres table in one transactional delete+insert.

- **Trigger:** manual
- **Nodes:** Postgres (read mapping + write results), Code (build JQL batches, build the reload SQL), Jira (search issues)

### [`TAXO_Completion_Notifier.json`](TAXO_Completion_Notifier.json) — Taxonomy request completion notifier
Runs every 30 minutes against a `taxo_requests` data table. For each request still pending, it checks whether the linked Jira ticket has closed, verifies in Postgres that the expected brand attributes actually landed in the warehouse, and then either emails the requester (done), escalates to an admin by email (closed >24h with data still missing), or waits for the next run.

- **Trigger:** schedule (every 30 min)
- **Nodes:** Data Table (read/update request status), Jira (search closed tickets), Postgres (verify brand data), Code (matching/decision logic), If, Email Send

### [`TAXO_Request_Intake_Form.json`](TAXO_Request_Intake_Form.json) — Taxonomy request intake form
A self-contained request form served entirely from n8n webhooks — no external hosting needed:
1. **GET** a webhook serves a generated HTML page (company/brand picker, built client-side in vanilla JS)
2. **GET** a second webhook acts as a brand-lookup API the page calls via `fetch()`, backed by a Postgres query
3. **POST** the submit webhook validates the selections, creates a Jira ticket with an auto-generated summary/description, and logs the request to a data table

Supports a `?dryRun=1` mode that previews exactly what would be sent to Jira without creating anything.

- **Trigger:** webhook (GET page, GET brands API, POST submit)
- **Nodes:** Webhook ×3, Data Table, Postgres, Code (HTML generation, request parsing/validation, dry-run preview), Jira (create issue), Respond to Webhook ×3, If (dry-run branch)

### [`AdTech_Biweekly_Read.json`](AdTech_Biweekly_Read.json) — AdTech biweekly digest
Every 14 days, pulls candidate stories from RSS feeds (trade press + Google News topic searches), a Gmail newsletter label, Reddit, and Hacker News, de-dupes and pre-scores them with keyword boosts, then has an LLM chain (AWS Bedrock) rank the top 6–8 against a client-integration-engineer persona before emailing a formatted HTML digest.

- **Trigger:** schedule (every 14 days)
- **Nodes:** Code (per-source normalization, candidate prep/scoring, email HTML build), RSS Feed Read, Gmail, HTTP Request (Reddit, Hacker News), Merge, LangChain LLM Chain + Bedrock Chat Model, Email Send

## Before you import

Every workflow in this repo has had the following stripped or genericized, since n8n exports embed whatever was in the original workflow:
- Credential references (`credentials` blocks) — these only resolve inside the original n8n instance anyway, so after importing you'll need to re-select or re-create each credential (Postgres, Jira, SMTP, AWS, Basic Auth, etc.)
- Internal data table / project IDs — nodes referencing a Data Table show `(re-select me)`; pick the matching table in your own instance
- Real company/person names used as illustrative examples in code comments — replaced with generic placeholders (e.g. `Acme Corp`)
- Hard-coded email addresses — replaced with `your.name@yourcompany.com` / `you@yourcompany.com` style placeholders; update these to your own addresses

## Setup

1. Import the `.json` file via n8n's **Import from File** (or paste into **Import from URL/Clipboard**)
2. Re-select/re-create every credential the workflow references (see each node's "re-select me" markers and the **Before you import** notes above)
3. Update any placeholder values (emails, base URLs, table/project selections) called out in that workflow's sticky notes
4. Test manually before activating a schedule- or webhook-triggered workflow
