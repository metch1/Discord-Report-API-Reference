# Discord Report API Reference

> **Disclaimer:** Unofficial breakdown of the message reporting API payload
> Written for anyone automating reports or digging into Trust & Safety flows

-------

## Overview

Discord’s message reporting system uses a **node-based navigation tree**. Each report type match to a unique path through that menu, the `breadcrumbs` array shows the steps, and `variant` is the final choice.

**HTTP Endpoint** `POST /api/v9/reporting/message`


----

## Payload Structure

| Field          | Type     | Description                                                                 |
|----------------|----------|-----------------------------------------------------------------------------|
| `version`      | string   | Always `"1.0"`                                                              |
| `variant`      | string   | ID of the **final node** in the report path (e.g., `"101"`)                 |
| `language`     | string   | Language code (e.g., `"en"`)                                                |
| `breadcrumbs`  | number   | Ordered list of node IDs leading to the final variant                       |
| `elements`     | object   | Always an empty object `{}` (future placeholder for additional metadata)    |
| `channel_id`   | string   | Discord channel ID where the reported message is located                    |
| `message_id`   | string   | Discord message ID to report                                                |
| `name`         | string   | Always `"message"` – resource type                                          |

----------

## Quick Reference

| Report Type        | Variant | Breadcrumbs            | Auto‑Submit | Internal Name     |
|--------------------|---------|------------------------|-------------|-------------------|
| Verbal Harassment  | `101`   | `[7, 76, 101]`         | No          |                   |
| Spam               | `98`    | `[7, 98]`              | No          |                   |
| Hate Speech        | `107`   | `[7, 76, 107]`         | No          |                   |
| Gore / Violence    | `108`   | `[7, 86, 108]`         | No          |                   |
| CSAM               | `116`   | `[7, 88, 116]`         | No          |                   |
| Scam / Fraud       | `167`   | `[7, 80, 121, 167]`    | No          | `MESSAGE_SCAM`    |
| Vulgar Language    | `103`   | `[7, 76, 103]`         | **Yes**     |                   |
| "Don’t Like"       | `97`    | `[7, 97]`              | **Yes**     |                   |

- **Auto‑Submit:** When `Yes`, the report is submitted **immediately** without additional confirmation prompts.
  When `No`, the user must confirm through one or more extra UI steps.

---

## Usage Examples

### C#

```csharp
using System.Text.Json;

var payload = new
{
    version = "1.0",
    variant = "101",
    language = "en",
    breadcrumbs = new[] { 7, 76, 101 },
    elements = new { },
    channel_id = "123456789123456789",
    message_id = "987654321987654321",
    name = "message"
};

string json = JsonSerializer.Serialize(payload);
```

```py
import json

payload = {
    "version": "1.0",
    "variant": "101",
    "language": "en",
    "breadcrumbs": [7, 76, 101],
    "elements": {},
    "channel_id": "123456789123456789",
    "message_id": "987654321987654321",
    "name": "message"
}

json_payload = json.dumps(payload)
```
## **Important Notes!**

1. **Auto‑submit** reports are **signal‑only** and don’t create tickets
   - Reports marked as Auto‑Submit (e.g., *Vulgar Language*, *“Don’t Like”*) are accepted immediately without a confirmation screen.
   - Internally these act as signals for automated moderation or statistics.
   - They **do not** generate a human-reviewed Trust & Safety ticket.
   - In contrast, reports like Harassment or CSAM require confirmation and do create tickets.

2. **Breadcrumbs** must match the exact navigation path Discord expects
   - The array represents the menu sequence a user would click.
   - Discord’s backend has a fixed tree – you cannot skip or reorder nodes.
   - Example: to report **Verbal Harassment** you must send `[7, 76, 101]` because:
     - `7` → “Report Message”
     - `76` → “Harassment”
     - `101` → “Verbal Harassment”
   - Any deviation (e.g., `[7, 101]`) is an invalid path and will be rejected.

3. **Variant** must equal the last element of **breadcrumbs**
   - `variant` is the destination node ID and must match the final breadcrumb.
   - For `[7, 76, 101]`, the variant must be `"101"`.
   - A mismatch (e.g., breadcrumbs ending in `101` but variant `"107"`) will cause a 400 error.

4. **Missing or incorrect breadcrumbs** result in `400 Bad Request`
   - Omitting the field, sending an empty array, or using a **non-existent** path all cause the API to reject the request.
   - The server validates the full path before processing, no report is created on failure.
