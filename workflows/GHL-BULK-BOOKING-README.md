# GHL Bulk Class Booking Workflow

**n8n Workflow ID:** `rqXGQsoCLwWbnhNk`  
**n8n Instance:** https://comebackperformancept.app.n8n.cloud  
**Workflow URL:** https://comebackperformancept.app.n8n.cloud/workflow/rqXGQsoCLwWbnhNk

---

## Overview

This workflow enables bulk booking of multiple appointments on the GHL Cheer Doctor Drop-In Calendar for a single contact. It presents a web form showing available time slots (with checkboxes), then books all selected slots in sequence via the GHL API.

## Webhook URLs

| Purpose | Method | URL |
|---|---|---|
| Get slot selection form | GET | `https://comebackperformancept.app.n8n.cloud/webhook/ghl-bulk-booking` |
| Submit booking selections | POST | `https://comebackperformancept.app.n8n.cloud/webhook/ghl-bulk-booking-submit` |

## How to Use

### Step 1 — Open the booking form

Make a GET request (or open in browser):

```
https://comebackperformancept.app.n8n.cloud/webhook/ghl-bulk-booking
  ?contactId=<GHL_CONTACT_ID>
  &startDate=<EPOCH_MS>
  &endDate=<EPOCH_MS>
  &timezone=America/New_York
```

**Query parameters:**

| Parameter | Required | Description |
|---|---|---|
| `contactId` | Yes | GHL contact ID to book appointments for |
| `startDate` | Yes | Start of date range as epoch milliseconds (e.g. `1746144000000`) |
| `endDate` | Yes | End of date range as epoch milliseconds (max 31 days from startDate) |
| `timezone` | No | Timezone for display (default: `America/New_York`) |

The response is an HTML page with checkboxes for each available slot. If no slots are available (e.g. calendar has no open hours configured), a **Manual Booking** textarea is shown instead — enter ISO datetime strings one per line.

### Step 2 — Select slots and submit

Check the desired slots and click **Book Selected Slots**. The form submits a POST to the second webhook automatically.

### Step 3 — View results

The form displays a summary: `Booked: N | Failed: N`. Each failed slot shows the error reason.

---

## Direct API Usage (Programmatic)

You can also POST directly to the submit webhook:

```bash
curl -X POST https://comebackperformancept.app.n8n.cloud/webhook/ghl-bulk-booking-submit \
  -H 'Content-Type: application/json' \
  -d '{
    "contactId": "GHL_CONTACT_ID",
    "timezone": "America/New_York",
    "selectedSlots": [
      {"startTime": "2026-05-10T10:00:00-04:00", "endTime": "2026-05-10T11:00:00-04:00"},
      {"startTime": "2026-05-11T14:00:00-04:00", "endTime": "2026-05-11T15:00:00-04:00"}
    ]
  }'
```

**Response:**
```json
[{
  "results": [
    {"startTime": "...", "endTime": "...", "success": true, "appointmentId": "abc123", "error": null},
    {"startTime": "...", "endTime": "...", "success": true, "appointmentId": "def456", "error": null}
  ],
  "total": 2,
  "successes": 2,
  "failures": 0
}]
```

---

## GHL Configuration

| Setting | Value |
|---|---|
| Calendar ID | `aEQMDSqBvV7BTesprnm8` |
| Calendar Name | Cheer Doctor Drop In Calendar |
| Calendar Type | class_booking |
| Location ID | `roM3u8ZgkU8ALWDRskJh` |
| Assigned User ID | `yteJJteIK1NEf1JM6og7` (Cheer Doctor Support) |
| Slot Duration | 60 minutes |
| Appointments per Slot | 8 |
| `ignoreFreeSlotValidation` | `true` (bypasses open hours check) |

---

## Workflow Architecture

```
[GET /ghl-bulk-booking]
  → Fetch GHL Free Slots (GET /calendars/{id}/free-slots)
  → Build Slot Selection Form (Code node → HTML)
  → Return HTML form (Respond to Webhook)

[POST /ghl-bulk-booking-submit]
  → Prepare Slot Items for Loop (Code node → N items)
  → Loop Through Slots (Split in Batches, batchSize=1)
    → Create GHL Appointment (POST /calendars/events/appointments)
    → Record Booking Result (Code node)
    → [loop back]
  → [loop done] Aggregate All Results
  → Return Booking Results (Respond to Webhook)
```

## Credential Setup

An **HTTP Header Auth** credential named `GHL Private Integration Token` is required:

- **Header Name:** `Authorization`
- **Header Value:** `Bearer pit-f28f2011-e59a-4e3d-8832-777428a5366b`

This credential is already configured on the n8n instance.
