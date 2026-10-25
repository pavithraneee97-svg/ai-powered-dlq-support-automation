# AI-Powered DLQ Support Automation

Claude skills that automate the daily support workflow for AWS SQS dead-letter queue (DLQ) messages: analyse failures, investigate logs in OpenSearch, and raise Jira tickets, with a human approving every action that changes something.

**Status:** Proposal. Designed and documented, not used in production.

## Overview

| Skill | Purpose |
|-------|---------|
| `support-activity` | Reads DLQ messages, compares them with already-tracked issues, investigates new ones in OpenSearch, and produces a summary table. On Mondays it also helps reprocess DLQ messages after user confirmation. |
| `jira-ticket-creation` | Takes issues marked **Needs Ticket** by `support-activity`, checks for duplicates, shows a preview, and creates Jira tickets only after the user approves. |

The two skills are chained: `support-activity` finds and explains the problem, `jira-ticket-creation` files it.

## Repository structure

```
.claude/
  skills/
    support-activity/
      SKILL.md
    jira-ticket-creation/
      SKILL.md
README.md
```

The helper scripts and tracking file used by the skills are **not stored in this repository** (see Prerequisites).

## Workflow

### Monday: reprocessing day
1. Run `read-dlq.py` to export all DLQ messages to a text file.
2. Summarise message counts per DLQ.
3. Ask the user which queues to reprocess (the `not-synced-values` queues are excluded).
4. After explicit confirmation, run `reprocess-dlq.py` for the chosen queues.
5. Report messages reprocessed, failures, and messages remaining.

Logs are not investigated on Mondays unless requested.

### Tuesday to Friday: investigation day
1. Run `read-dlq.py` to export all DLQ messages.
2. Compare message IDs with `support-ticket.txt`. IDs already listed are skipped.
3. For each new ID, search OpenSearch for the last 50 hours, find the session ID, then pull the ERROR logs for that session.
4. Collect error message, stack trace, timestamp, service, exception class and request URL.
5. Output a table with the status **Existing Ticket** or **Needs Ticket**.

### Ticket creation (after investigation)
1. Select only rows marked **Needs Ticket**.
2. Check for duplicates in `support-ticket.txt` and in Jira.
3. Build the ticket from the standard template.
4. Show a preview and wait for an explicit "yes".
5. Create the ticket, then append `<ID>,<DLQ name>,<JIRA KEY>` to `support-ticket.txt`.
6. Report the result for each ticket.

## Prerequisites

- Claude Code (or Claude with skills support)
- Python 3 and the local scripts `read-dlq.py` and `reprocess-dlq.py`
- `support-ticket.txt` in the same folder as the scripts, one line per ticket: `ID,DLQ,JIRA-KEY`
- AWS CLI profile with **read-only** access for DLQ and log investigation
- Network access to the OpenSearch endpoint
- Atlassian (Jira) MCP server connected (check with `/mcp`)

## Configuration

Set these values in each `SKILL.md` locally. Do not commit real values.

| Setting | Skill | Description |
|---------|-------|-------------|
| AWS profile | `support-activity` | Read-only profile name |
| OpenSearch URL | `support-activity` | Search endpoint |
| Jira site | `jira-ticket-creation` | `<your-company>.atlassian.net` |
| Jira project key | `jira-ticket-creation` | Project to create tickets in |
| Default labels / priority | `jira-ticket-creation` | `dlq, production-support` / Medium |

## Usage

Open Claude Code in the project folder and ask, for example:

- "Run the support-activity skill for today."
- "Create Jira tickets for the issues marked Needs Ticket."

## Safety rules

- Never delete DLQ messages.
- Never reprocess messages automatically. Reprocessing happens on Mondays, after user confirmation, unless the user asks otherwise.
- AWS access is read-only. No write operations with the AWS profile.
- `support-activity` never creates Jira tickets.
- `jira-ticket-creation` never creates a ticket without an explicit "yes", and always runs both duplicate checks.
- `support-ticket.txt` is append-only. Existing lines are never edited or removed.
- No credentials, tokens, internal hostnames, or customer personal data in tickets or in this repository.
- If a Jira call fails, the skill shows the error and stops instead of retrying.
