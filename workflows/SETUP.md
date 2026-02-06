# ConnectWise PSA - Nutanix Restore Validation Workflow

## Overview

This workflow polls ConnectWise Manage (via `portal.choicecloud.com`) for restore-related tickets, scans their notes for status keywords, and verifies restored VMs exist on Nutanix Prism Central when a restore is marked as completed.

## Workflow Flow

```
Schedule Trigger (every 15 min)
    │
    ▼
Query CW Restore Tickets
  GET /service/tickets
  conditions: summary contains 'Restore' AND status not in (Cancelled, Closed, Completed)
  Auth: CWM Header API Auth (RS1aHxghqa9C5VAg)
  clientId: 0d5da4ac-da1e-4362-b723-881330675fe4
    │
    ▼
Split Tickets (iterate each)
    │
    ▼
Get Ticket Notes
  GET /service/tickets/{id}/notes
  Same auth + clientId
    │
    ▼
Scan Notes for Keywords (Code node)
  Keywords: completed, restore completed, validated, submitted
    │
    ▼
IF Restore Completed?
    │
   YES ──────────────────────────────┐
    │                                ▼
   NO                     Query Nutanix VMs
    │                     POST /api/nutanix/v3/vms/list
    ▼                     Auth: HTTP Basic (Nutanix admin)
IF Validated/Submitted?              │
    │                                ▼
   YES → In Progress          Parse Nutanix VM Results
    │                                │
   NO → No Keywords                  ▼
                              IF VM Found?
                               │        │
                              YES       NO
                               │        │
                               ▼        ▼
                          VM Verified  VM Not Found
```

## What's Already Configured (from your CW Manage MCP Agent)

The ConnectWise side is **ready to go** — it uses the same auth pattern as your existing workflow:

| Setting | Value |
|---|---|
| **CW API Host** | `https://portal.choicecloud.com/v4_6_release/apis/3.0/` |
| **Auth Type** | HTTP Header Auth |
| **Credential ID** | `RS1aHxghqa9C5VAg` ("CWM Header API Auth") |
| **clientId Header** | `0d5da4ac-da1e-4362-b723-881330675fe4` |

## What You Need to Set Up (Nutanix Only)

### 1. Environment Variable

Set this in n8n under **Settings > Environment Variables**:

| Variable | Description | Example |
|---|---|---|
| `NUTANIX_PRISM_HOST` | Nutanix Prism Central IP or hostname | `10.0.0.50` or `prism.example.com` |

### 2. Nutanix Credential (HTTP Basic Auth)

Create a new credential in n8n:

1. Go to **Credentials** > **Add Credential**
2. Choose **HTTP Basic Auth**
3. Enter your Nutanix Prism Central admin username and password
4. Save it and note the credential ID

Then in the workflow JSON, find the `"Query Nutanix VMs"` node and replace:
```json
"id": "NUTANIX_BASIC_AUTH_CREDENTIAL_ID"
```
with your actual credential ID, e.g.:
```json
"id": "aBcDeFgHiJkLmNoP"
```

## Import Instructions

1. Open your n8n instance
2. Go to **Workflows** > **Add Workflow** > **...** menu > **Import from File**
3. Select `connectwise-nutanix-restore-validation.json`
4. Update the Nutanix credential ID (see above)
5. Set the `NUTANIX_PRISM_HOST` environment variable
6. Save and activate

## Keyword Detection Logic

The Code node scans all ticket notes for these keywords (case-insensitive):

| Keyword | Status Category | Action |
|---|---|---|
| `restore completed` or `completed` | `completed` | Proceeds to Nutanix VM verification |
| `validated` | `validated` | Logs as in-progress, no Nutanix check |
| `submitted` | `submitted` | Logs as in-progress, no Nutanix check |
| _(none found)_ | `no_match` | Logs as no action needed |

Priority order: `completed` > `validated` > `submitted`

## VM Name Pattern

The workflow expects restored VMs on Nutanix to include `Restore` and a date in their name:

- `ServerName_Restore_2026-02-06`
- `DB-Server-Restore-2026-02-06`
- `WebApp_Restore_20260206`

## Customization

- **Polling interval:** Edit the Schedule Trigger (default: 15 minutes)
- **Ticket filter:** Modify the `conditions` query param in "Query CW Restore Tickets"
- **Keywords:** Edit the `keywords` array in the "Scan Notes for Keywords" Code node
- **VM name regex:** Adjust the pattern in "Parse Nutanix VM Results" Code node
