---
name: support-activity
description: Daily support assistant for DLQ analysis, ticket verification, log investigation and Monday reprocessing.
---

# Parameters


AWS Profile =>  staging - ONLY READ ACCESS
AWS openserach URL => 
# Purpose

This skill automates the daily production support workflow for AWS DLQ messages.

The workflow changes depending on the day of the week.

## Available Local Scripts

scripts will be available within <project-folder>/support-activity/daily

Running python script 'read_dlq.py' will create dlq messages text file within the same directory. 

```bash
python3 ./read_dlq.py
```

Running python script 'reprocess_dlq.py' will prompt user for which dlq number and client queue to reprocess. Arguments can also be passed to 
reprocess particualr queue. 

```bash
python3 ./reprocess-dlq.py 
```

These scripts already exist.
Do not modify them.

---

# Monday Workflow

On Mondays only:

1. Execute

```
python3 ./read-dlq.py
```

2. Wait for sqs_dlq_message_date time text file to be created within the same directory, it contains all the dlq messages.

3. Produce a summary

Example

DLQ | Count
----|------
dlq-name | 14

4. Ask user which queue should be reprocessed except production-not-synced-values/ staging-not-synced-values. Get confirmation to reprocess 
all the dlq messages in  a single attempt except production-not-synced-values/ staging-not-synced-values.

Never automatically reprocess.

Example
Queue: client-1019-find-to-save-resource-dlq
Message Count: 2
Messages:
{"resourceType":"DOCUMENT","id":"528316","caseId":null,"additionalInfo":null}
{"resourceType":"DOCUMENT","id":"526835","caseId":null,"additionalInfo":null}


5. Execute

```
#  python3 ./reprocess-dlq.py queue-number queue-name client-id
python3 ./reprocess-dlq.py 2 1 1019
```

6. Display

- Number of messages reprocessed
- Failed reprocesses
- Remaining messages

Do not investigate logs on Monday unless requested.

---

# Tuesday-Friday Workflow

1. Execute

```
python3 ./read-dlq.py
```
2. Wait for sqs_dlq_message_date time text file to be created within the same directory, it contains all the dlq messages.

3. Read

```
support-ticket.txt
```

The file contains

ID
DLQ
JIRA Ticket

Example

12345,RecordDLQ,ABC-123
88888,InvoiceDLQ,ABC-140

4. Compare

If an ID already exists in support-ticket.txt

Skip.

Otherwise

Treat it as a new support issue.

---

# New Support Issue Investigation

For every new ID

Search OpenSearch (URL provided) for the past 50 hours and get into the particular session.

Search format

1) Search in ES using below format. the queue name must be the queue name instead of dlq name. Map is provided at the beginning of this file. and filter the response based on the timestamp.
```
 curl -s ES URL \
    -H "Content-Type: application/json" \
    -d '{
      "size": 30,
      "query": {
        "query_string": {"query": "Whole AND \"<ID>\" AND \"<QUEUE_NAME>\""}
      }
    }'

```

2) Once result is returned, find the session id asssociated with that and filter it again. See step 3.
```
 curl -s ES URL \
    -H "Content-Type: application/json" \
    -d '{
      "size": 30,
      "query": {
        "query_string": {"query": "Whole AND \"<ID>\" AND \"<QUEUE_NAME>\""}
      }
    }'

```

3) Investigate based on the session id
```
 curl -s ES URL \
    -H "Content-Type: application/json" \
    -d '{
      "size": 10,
      "query": {
        "bool": {
          "must": [
            {"term": {"session": "<SESSION_ID>"}},
            {"term": {"severity": "ERROR"}}
          ]
        }
      }
    }'

```

Example

Whole AND "123456" AND "client-1019-wa-resources-to-sa-queue"

Collect

- Error message
- Stacktrace (if available)
- Timestamp
- Service
- Exception class
- Request URL

Summarize

Example

ID:
123456

DLQ:
RecordDLQ

Root Cause:
NullPointerException

Failure:
Invoice reference missing

Service:
record-service

Time:
10:41 UTC

Suggested Action:
Needs application fix.

---

# Output

Produce a markdown table

| ID | DLQ | Root Cause | JIRA | Status |
|----|------|------------|------|--------|

Status

Existing Ticket

or

Needs Ticket

Never create Jira tickets automatically.

That is handled by another skill.

---

# Rules

Never delete messages.

Never reprocess except Monday unles notified

Never modify support-ticket.txt.

Never perform WRITE operations with aws profile

Always summarize findings before asking for next action.
