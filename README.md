# AI Appointment Setter

An n8n workflow that gives a **Vapi AI voice agent** full control over Google Calendar — booking, rescheduling, and cancelling appointments in real time during phone calls. All activity is logged to Airtable for tracking and analytics.

---

## How It Works

The Vapi voice agent makes outbound or receives inbound calls. During the conversation it calls n8n **webhook tool endpoints** to take calendar actions. n8n connects to Google Calendar and Airtable, then returns the result back to the agent so it can respond naturally to the caller.

```
Caller → Vapi AI Agent → n8n Webhook → Google Calendar / Airtable → Response back to Vapi
```

---

## Workflow Sections

### 1. Get Slots (`POST /webhook/getslots`)

**Purpose:** Check whether a proposed time slot is available, and if not, suggest alternatives.

**Flow:**
1. Vapi sends the proposed `starttime` and `endtime` from the tool call.
2. **Code Node** extracts the arguments from the Vapi payload.
3. **Set Node** maps timezone and tool call ID.
4. **Google Calendar – Check Availability** queries the calendar for that time range.
5. **IF: IsTimeAvailable?**
   - **True (available):** Responds immediately with `available:true`.
   - **False (busy):** Fetches all events for the next week → extracts start/end/name → sorts chronologically → aggregates → runs a **Code node** that calculates 30-minute available slots and "wide open ranges" → responds with the list so the agent can offer alternatives.

---

### 2. Book Slots (`POST /webhook/bookslots`)

**Purpose:** Create a Google Calendar event and log the appointment to Airtable.

**Flow:**
1. Vapi sends `starttime`, `endtime`, `email`, `name`, `Title`, and `notes`.
2. **Code Node** extracts the arguments from the Vapi payload.
3. **Set Node** maps all booking fields and the tool call ID.
4. **IF: Has All Information?** — validates that `email` (must be a valid email address), `name`, and `notes` are present.
   - **Missing fields:** Builds an error response telling the agent which fields are required → responds to Vapi → ends.
   - **All present:** Continues.
5. **Escape JSON** — sanitises the `notes` field to prevent JSON injection.
6. **Convert Time to CST** — converts UTC ISO timestamps from Vapi to `America/Chicago` (CST/CDT).
7. **Google Calendar – Create Event** — creates the event with attendees, Google Meet link, description, and summary.
   - **On error:** Runs **Add Friendly Error** to humanise the error message → builds an error response → responds to Vapi.
   - **On success:** Extracts the booking payload (event ID, status, hangout link, attendees, times).
8. **Success Response** — formats the `confirmed` status back to Vapi.
9. **Respond to Vapi** — sends the response and continues.
10. **ConfirmedBooking? (Filter)** — only proceeds if the Google Calendar status is `confirmed`.
11. **Log Book Details (Airtable)** — saves Email, Phone Number, Name, Booking Status, Call Recording ID, start/end times, Meet link, description, Voice Agent name, and Google event ID to the `Appointments` table.

---

### 3. Update Slots (`POST /webhook/updateslots`)

**Purpose:** Reschedule an existing appointment to a new time.

**Flow:**
1. Vapi sends `starttime` (original), `endtime` (original), `Rescheduled_starttime`, `Rescheduled_endtime`, `email`, and `name`.
2. **Code Node** extracts all arguments from the Vapi payload.
3. **Set Node** maps fields including the original start/end and the new rescheduled times.
4. **IF: Check if All Required Info is Present** — validates name, email, original times, and rescheduled times.
   - **Missing:** Builds an error response → responds to Vapi → ends.
   - **All present:** Continues.
5. **Find Original Appointment (Airtable)** — searches the `Appointments` table by the caller's phone number to retrieve the original Google event ID.
6. **Google Calendar – Update Event** — moves the event to the rescheduled start/end time.
   - **On error:** Passes to **Response and Call_id** with the error.
   - **On success:** Also passes to **Response and Call_id**.
7. **Update Record (Airtable)** — updates the `Appointments` row: sets Booking Status to `Updated / Rescheduled` and updates start/end times. Matched by `eventId`.
8. **Response and Call_id** — formats the result (status or error) for Vapi.
9. **Respond to Vapi update slots results** — sends the final response.

---

### 4. Cancel Slots (`POST /webhook/cancelslots`)

**Purpose:** Delete an existing Google Calendar event and mark it cancelled in Airtable.

**Flow:**
1. Vapi sends `name`, `email`, `starttime`, and optional `Cancelnotes`.
2. **Code Node** extracts all arguments from the Vapi payload.
3. **Set Node** maps fields including the tool call ID.
4. **IF: Check if Required Info is Provided** — validates name, email, and starttime.
   - **Missing:** Builds an error response → responds to Vapi → ends.
   - **All present:** Continues.
5. **Find Appointment Record (Airtable)** — searches the `Appointments` table by phone number to get the Google event ID.
6. **Google Calendar – Delete Event** — deletes the event and sends cancellation notices to all attendees.
   - On error output, flow continues (gracefully handles "event not found").
7. **Update Record to Cancelled (Airtable)** — sets Booking Status to `Cancelled` in the matched `eventId` row.
8. **Call_id and Response** — formats the cancellation result for Vapi.
9. **Respond to Vapi about cancellation** — sends the final response.

---

### 5. Call Results (`POST /webhook/callresults`)

**Purpose:** Receive the call-end event from Vapi and log full call details to Airtable.

**Flow:**
1. Vapi sends the call-end webhook containing transcript, recording URL, call summary, cost, assistant info, and caller number.
2. **All Input Arguments (Set Node)** — extracts and maps all relevant fields.
3. **Create or Update a Record (Airtable – `Call Recording` table)** — upserts by `callrecording_id`: saves recording URL, transcript, call summary, cost, start/end times, and customer number.

---

## Required Credentials

| Service | Credential Type | Used In |
|---|---|---|
| Google Calendar | `googleCalendarOAuth2Api` | All calendar operations |
| Airtable | `airtableTokenApi` | Appointments + Call Recording tables |

Configure these in your n8n instance under **Settings → Credentials** before activating the workflow.

---

## Airtable Setup

**Base ID:** `your-airtable-base-id` — *AI Appointment Setter Build*

### Table: `Appointments`
| Field | Type | Description |
|---|---|---|
| Email | string | Attendee email |
| Phone Number | string | Caller's number (used as lookup key) |
| Name | string | Attendee name |
| Booking Status | string | `confirmed`, `Updated / Rescheduled`, `Cancelled` |
| CallRecordingId | array | Linked call recording |
| starttime | dateTime | Appointment start |
| endtime | dateTime | Appointment end |
| meetlink | string | Google Meet URL |
| meetdescription | string | Title + notes |
| Voice Agent | array | Vapi assistant name |
| eventId | string | Google Calendar event ID (match key for updates/cancels) |

### Table: `Call Recording`
| Field | Type | Description |
|---|---|---|
| callrecording_id | string | Vapi call ID (upsert key) |
| Cost | number | Vapi call cost |
| Call recording Url | string | Recording audio URL |
| transcript | string | Full call transcript |
| customer_Number | string | Caller's phone number |
| startedAt | dateTime | Call start time |
| endedAt | dateTime | Call end time |
| callsummary | string | AI-generated call summary |

---

## Vapi Tool Configuration

Register these as **Server URL tools** in your Vapi assistant:

| Tool Name | Method | URL |
|---|---|---|
| `getslots` | POST | `https://your-n8n-instance.com/webhook/getslots` |
| `bookslots` | POST | `https://your-n8n-instance.com/webhook/bookslots` |
| `updateslots` | POST | `https://your-n8n-instance.com/webhook/updateslots` |
| `cancelslots` | POST | `https://your-n8n-instance.com/webhook/cancelslots` |

Set the **call-end webhook** URL to:
```
POST https://your-n8n-instance.com/webhook/callresults
```

### Tool Arguments Reference

**getslots**
- `starttime` (string) — ISO 8601 UTC start of proposed slot
- `endtime` (string) — ISO 8601 UTC end of proposed slot

**bookslots**
- `starttime` (string) — ISO 8601 UTC
- `endtime` (string) — ISO 8601 UTC
- `email` (string, required) — attendee email
- `name` (string, required) — attendee name
- `Title` (string) — event title
- `notes` (string, required) — agenda / description

**updateslots**
- `starttime` (string) — original appointment start
- `endtime` (string) — original appointment end
- `Rescheduled_starttime` (string) — new start time
- `Rescheduled_endtime` (string) — new end time
- `email` (string) — attendee email
- `name` (string) — attendee name

**cancelslots**
- `starttime` (string) — appointment start to cancel
- `email` (string) — attendee email
- `name` (string) — attendee name
- `Cancelnotes` (string) — optional cancellation reason

---

## Configuration Notes

- **Timezone:** All bookings are converted to `America/Chicago` (CST/CDT). Change the timezone string in the **Convert Time to CST** and **Available Start Times & Ranges** Code nodes if needed.
- **Working hours:** The availability algorithm uses 9 AM – 6 PM CST, Monday–Friday. Adjust `WORKDAY_START` and `WORKDAY_END` in the **Available Start Times & Ranges** Code node.
- **Slot duration:** 30-minute slots. Change `SLOT_DURATION` in the same Code node.
- **Calendar:** Update the `calendar.value` in all Google Calendar nodes to your calendar ID (e.g. `your-email@gmail.com`).

---

