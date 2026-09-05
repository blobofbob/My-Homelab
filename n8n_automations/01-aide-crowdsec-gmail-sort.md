# 01 — AIDE/CrowdSec Gmail Sort
Welcome to the first n8n automation guide of this series. these are completely optional and might not even be useful to you if you 
dont have the mentioned services installed.

An n8n workflow that watches the inbox for AIDE and CrowdSec notification emails, labels each by
source, and archives them out of the inbox — AIDE mail stays marked **unread** as a quick visual
flag to check it, CrowdSec mail is marked **read** and kept purely as a log backup.

**Prerequisites:** [09 — OpenClaw + n8n](../guides/09-openclaw-n8n.md) complete (n8n running,
reachable at `https://raspberrypi.tail7aae4f.ts.net:8444`). A Gmail OAuth2 credential already set up
in n8n. Two nested Gmail labels already created: `homelab notifs/aide` and
`homelab notifs/crowdsec`.

---

## Why this exists

CrowdSec ([15 — CrowdSec](../guides/15-crowdsec.md)) sends a notification email on every ban
decision — useful as a record, not useful as something to read individually. AIDE
([13 — AIDE](../guides/13-aide.md)) sends a nightly integrity-check report via root's cron
`MAILTO`, which is lower-volume but does need at least a glance most days. Both were piling up
unsorted in the inbox. This workflow splits them by source and gets them out of the way
automatically, while still preserving them (as opposed to deleting) for later reference.

---

## How it works

1. **Gmail Trigger** polls every 1 minute, scoped with a Gmail search filter so it only ever fires
   on relevant mail in the first place:
   ```
   in:inbox subject:(aide OR "CrowdSec Notification")
   ```
2. **Switch node** routes on `{{$json.subject}}`:
   - contains `aide` (case-sensitive, matches AIDE's cron-generated subject line) → AIDE branch
   - equals `CrowdSec Notification` exactly → CrowdSec branch
3. Each branch runs three Gmail API calls in sequence:
   - **Add label** (`homelab notifs/aide` or `homelab notifs/crowdsec`)
   - **Mark as unread** (AIDE branch) / **Mark as read** (CrowdSec branch) — the deliberate
     asymmetry: AIDE mail is left visually flagged since it's better to have a quick look at each one;
     CrowdSec mail doesn't need review, so it's marked read on the way to the archive.
   - **Remove label `INBOX`** — this is what actually archives the message in Gmail; removing the
     `INBOX` label is functionally identical to clicking Archive.

> **The workflow must be toggled Active, not just saved, for the Gmail Trigger to start polling.**
> Saving alone does not start it — this caused an earlier "it doesn't work" report before the
> Publish/Active toggle was flipped.

---

## Workflow JSON

Import via n8n → click on **Create Workflow** and paste the following code block.

```json
{
  "name": "Homelab Notifications - Sort AIDE/CrowdSec",
  "nodes": [
    {
      "parameters": {
        "pollTimes": {
          "item": [
            {
              "mode": "everyX",
              "value": 1,
              "unit": "minutes"
            }
          ]
        },
        "simple": false,
        "filters": {
          "q": "in:inbox subject:(aide OR \"CrowdSec Notification\")"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.gmailTrigger",
      "typeVersion": 1.2,
      "position": [-128, 128],
      "name": "Gmail Trigger",
      "id": "479637ca-eeeb-4081-b52e-61dfebaee7be",
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_YOUR_CRED_ID",
          "name": "Gmail account"
        }
      }
    },
    {
      "parameters": {
        "rules": {
          "values": [
            {
              "conditions": {
                "options": {
                  "caseSensitive": true,
                  "leftValue": "",
                  "typeValidation": "strict",
                  "version": 2
                },
                "conditions": [
                  {
                    "leftValue": "={{$json.subject}}",
                    "rightValue": "aide",
                    "operator": {
                      "type": "string",
                      "operation": "contains"
                    },
                    "id": "15a0ff1d-db88-438f-8ec2-6309bc0a5336"
                  }
                ],
                "combinator": "and"
              }
            },
            {
              "conditions": {
                "options": {
                  "caseSensitive": true,
                  "leftValue": "",
                  "typeValidation": "strict",
                  "version": 2
                },
                "conditions": [
                  {
                    "leftValue": "={{$json.subject}}",
                    "rightValue": "CrowdSec Notification",
                    "operator": {
                      "type": "string",
                      "operation": "equals"
                    },
                    "id": "766936bc-d298-4bd4-bea2-954ad596b652"
                  }
                ],
                "combinator": "and"
              }
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.switch",
      "typeVersion": 3.2,
      "position": [96, 128],
      "name": "Route by Subject",
      "id": "533549b6-f135-45e2-bbcc-1707093688b3"
    },
    {
      "parameters": {
        "operation": "addLabels",
        "messageId": "={{$json.id}}",
        "labelIds": ["Label_5599770736887633811"]
      },
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [320, 32],
      "name": "Label - AIDE",
      "id": "fb2b4528-3904-4f47-b1c9-389254c6da69",
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_YOUR_CRED_ID",
          "name": "Gmail account"
        }
      }
    },
    {
      "parameters": {
        "operation": "markAsUnread",
        "messageId": "={{$json.id}}"
      },
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [544, 32],
      "name": "Archive - AIDE",
      "id": "aba0a7d0-6e52-4a15-aa88-842abf65aac1",
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_YOUR_CRED_ID",
          "name": "Gmail account"
        }
      }
    },
    {
      "parameters": {
        "operation": "addLabels",
        "messageId": "={{$json.id}}",
        "labelIds": ["Label_6918479611429248589"]
      },
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [320, 224],
      "name": "Label - CrowdSec",
      "id": "0aa5e6cf-a78e-4da6-9a80-d350dc1abda0",
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_YOUR_CRED_ID",
          "name": "Gmail account"
        }
      }
    },
    {
      "parameters": {
        "operation": "markAsRead",
        "messageId": "={{$json.id}}"
      },
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [544, 224],
      "name": "Archive - CrowdSec",
      "id": "7a46fbbe-2200-4deb-8ac6-c96b19ccf3c9",
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_YOUR_CRED_ID",
          "name": "Gmail account"
        }
      }
    },
    {
      "parameters": {
        "operation": "removeLabels",
        "messageId": "={{$json.id}}",
        "labelIds": ["INBOX"]
      },
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.2,
      "position": [768, 224],
      "id": "c38940da-2cae-452a-8a3b-56c8b760b87a",
      "name": "Remove label from message",
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_YOUR_CRED_ID",
          "name": "Gmail account"
        }
      }
    },
    {
      "parameters": {
        "operation": "removeLabels",
        "messageId": "={{$json.id}}",
        "labelIds": ["INBOX"]
      },
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.2,
      "position": [768, 32],
      "id": "dd66dfd0-8861-4c79-b06e-308e3fcbda03",
      "name": "Remove label from message1",
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_YOUR_CRED_ID",
          "name": "Gmail account"
        }
      }
    }
  ],
  "connections": {
    "Gmail Trigger": {
      "main": [[{ "node": "Route by Subject", "type": "main", "index": 0 }]]
    },
    "Route by Subject": {
      "main": [
        [{ "node": "Label - AIDE", "type": "main", "index": 0 }],
        [{ "node": "Label - CrowdSec", "type": "main", "index": 0 }]
      ]
    },
    "Label - AIDE": {
      "main": [[{ "node": "Archive - AIDE", "type": "main", "index": 0 }]]
    },
    "Label - CrowdSec": {
      "main": [[{ "node": "Archive - CrowdSec", "type": "main", "index": 0 }]]
    },
    "Archive - CrowdSec": {
      "main": [[{ "node": "Remove label from message", "type": "main", "index": 0 }]]
    },
    "Archive - AIDE": {
      "main": [[{ "node": "Remove label from message1", "type": "main", "index": 0 }]]
    }
  },
  "active": true,
  "settings": {
    "executionOrder": "v1",
    "binaryMode": "separate"
  }
}
```

> **Credential and label IDs are stripped/placeholder'd above.** After import, reselect your Gmail
> OAuth2 credential on all five Gmail-type nodes, and re-pick the correct label from the dropdown on
> `Label - AIDE` and `Label - CrowdSec` (label IDs are account-specific and won't resolve from
> someone else's export).

---

## Setup

1. Confirm both nested labels exist in Gmail: `homelab notifs/aide`, `homelab notifs/crowdsec`.
2. n8n → **Create Workflow** → paste the JSON above.
3. Reselect the Gmail OAuth2 credential on all five Gmail nodes (trigger + 4 message-action nodes).
4. Open `Label - AIDE` and `Label - CrowdSec`, click into the `Label` field, and pick the correct
   label from the dropdown.
5. **Activate** the workflow (top-right toggle, **publish**) — saving alone does not start the trigger.

---

## Verifying

Send a throwaway test email to the watched inbox with subject `Cron <root@raspberrypi> test aide`
to confirm the AIDE branch fires, and one with subject exactly `CrowdSec Notification` for the
CrowdSec branch — faster than waiting for a real nightly AIDE run or CrowdSec ban to catch a
routing bug.

Check in Gmail:
- AIDE test mail → labeled `homelab notifs/aide`, out of the inbox, still shown as **unread**.
- CrowdSec test mail → labeled `homelab notifs/crowdsec`, out of the inbox, shown as **read**.

---

## Troubleshooting

**Workflow is saved but nothing happens**

Confirm it's actually **Active** by publishing the workflow (button on the top right of the workflow), not just saved — the Gmail Trigger only starts polling once toggled on.

---

**A test email doesn't get routed to either branch**

Check the exact subject against the Switch node's conditions — the AIDE match is
case-sensitive `contains "aide"`, the CrowdSec match is case-sensitive `equals "CrowdSec
Notification"` exactly. A subject that doesn't match either falls through with no output.

---

**Mail gets labeled but doesn't leave the inbox**

The `removeLabels: INBOX` step (`Remove label from message` / `Remove label from message1`) runs
after the mark read/unread step in each branch — confirm both nodes ran in the execution log, not
just the label-add step.

---

**Next:** TBD — a second automation is planned for this directory.
