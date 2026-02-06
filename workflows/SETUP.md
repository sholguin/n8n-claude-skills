# ConnectWise PSA - Nutanix Restore Validation Workflow

## Overview

This workflow polls ConnectWise PSA for restore-related tickets, scans their notes for status keywords, and verifies restored VMs exist on Nutanix when a restore is marked as completed.

## Workflow Flow

```
Schedule Trigger (every 15 min)
    |
    v
Query CW Tickets (GET /service/tickets where summary contains "Restore")
    |
    v
Split Tickets (iterate each ticket)
    |
    v
Get Ticket Notes (GET /service/tickets/{id}/notes)
    |
    v
Scan Notes for Keywords (Code node - scans for: completed, restore completed, validated, submitted)
    |
    v
IF Restore Completed? ----YES----> Query Nutanix VMs (POST /api/nutanix/v3/vms/list)
    |                                       |
    NO                                      v
    |                               Parse Nutanix VM Results
    v                                       |
IF Validated/Submitted?                     v
    |                               IF VM Found?
   YES --> In Progress Output         |         |
    |                                YES        NO
   NO --> No Keywords Output          |         |
                                      v         v
                              VM Verified   VM Not Found
                                Output        Output
```

## Required Environment Variables

Set these in n8n under Settings > Environment Variables:

| Variable | Description | Example |
|---|---|---|
| `CONNECTWISE_HOST` | ConnectWise PSA API hostname | `na.myconnectwise.net` |
| `CONNECTWISE_CLIENT_ID` | ConnectWise API Client ID (from developer portal) | `your-client-id-uuid` |
| `NUTANIX_HOST` | Nutanix Prism Central IP or hostname | `10.0.0.50` or `prism.example.com` |

## Required Credentials

### 1. ConnectWise PSA API Auth (HTTP Basic Auth)

Create an HTTP Basic Auth credential in n8n with:
- **Username:** `companyId+publicKey` (e.g., `mycompany+AbCdEfGh`)
- **Password:** Your ConnectWise API private key

Then update the credential ID in the workflow JSON:
- Replace `CONNECTWISE_BASIC_AUTH_CREDENTIAL_ID` with your actual n8n credential ID

### 2. Nutanix Prism Central Auth (HTTP Basic Auth)

Create an HTTP Basic Auth credential in n8n with:
- **Username:** Your Nutanix admin username
- **Password:** Your Nutanix admin password

Then update the credential ID in the workflow JSON:
- Replace `NUTANIX_BASIC_AUTH_CREDENTIAL_ID` with your actual n8n credential ID

## Import Instructions

1. Open your n8n instance
2. Go to **Workflows** > **Add workflow** (or use the import option)
3. Click the **...** menu > **Import from File**
4. Select `connectwise-nutanix-restore-validation.json`
5. Update the two credential references with your actual credential IDs
6. Set the required environment variables
7. Save and activate the workflow

## Keyword Detection Logic

The Code node scans all ticket notes (concatenated) for these keywords:

| Keyword | Status Category | Action |
|---|---|---|
| `restore completed` or `completed` | `completed` | Proceeds to Nutanix VM verification |
| `validated` | `validated` | Logs as in-progress, no Nutanix check |
| `submitted` | `submitted` | Logs as in-progress, no Nutanix check |
| _(none found)_ | `no_match` | Logs as no action needed |

Priority: `completed` > `validated` > `submitted`

## VM Name Pattern

The workflow expects restored VMs on Nutanix to follow a naming convention containing:
- The keyword `Restore` in the VM name
- A date stamp (e.g., `2026-02-06`)

Examples of expected VM names:
- `ServerName_Restore_2026-02-06`
- `DB-Server-Restore-2026-02-06`
- `WebApp_Restore_20260206`

## Customization

- **Polling interval:** Change the Schedule Trigger from 15 minutes to your preference
- **Ticket query filter:** Modify the `conditions` parameter in "Query CW Tickets" to match your ticket naming convention
- **Keywords:** Edit the `keywords` array in the "Scan Notes for Keywords" Code node
- **VM name pattern:** Adjust the regex in "Parse Nutanix VM Results" to match your naming convention
