# MetaAds → WhatsApp → Claude → Appointment (n8n)

An n8n workflow that turns a **Meta (Facebook/Instagram) Lead Ads** submission into a real
WhatsApp conversation run by **Claude**, qualifies the lead, and books a consultation or site
visit — writing every step back to a Google Sheets CRM.

Built for an interior-design business (Ascent Interiors, Kolkata), but the structure works for
any lead-gen → appointment funnel. Only the system prompt is business-specific.

```
Meta Lead Ad
   ↓
Webhook (GET verify + POST lead)
   ↓  fast 200 to Meta
Meta Graph API → full lead details
   ↓
Normalize phone → E.164
   ↓
Lookup in CRM ──┬── found ──→ load conversation state ──┐
                └── new   ──→ create row + init state ──┤
                                                        ↓
                                          Generate AI Reply ←── Claude
                                                        ↓
                                              Parse + validate
                                                        ↓
                        ┌─────────────── route ───────────────┐
                        ↓                ↓                    ↓
                 check availability   book appointment    plain reply
                        ↓                ↓                    ↓
                        └──────→ Send WhatsApp ←──────────────┘
                                        ↓
                                 Record delivery
                                        ↓
                                   Update CRM
                                        ↓
                          booking confirmed? → WhatsApp confirmation
```

**36 nodes · 33 connections · n8n 2.22.6**

---

## What each part does

### 1. Meta webhook (`Webhook` → `If1`)
One webhook handles both Meta requirements:

- **GET** — Meta's subscription check. `If1` compares `hub.verify_token` against
  `META_VERIFY_TOKEN` and echoes `hub.challenge` back. A wrong token gets nothing.
- **POST** — a real lead event. `Ack Meta 200` replies immediately so Meta never waits on the
  AI, WhatsApp or booking calls. Meta retries anything slow, which is how you get duplicates.

### 2. Getting the actual lead (`Extract Lead Event` → `Meta Get Lead Details`)
Meta's webhook is deliberately thin — it carries a `leadgen_id` and some ad IDs, **no personal
data**. The workflow calls the Graph API with that ID to fetch name, phone, email, campaign, ad,
form and every custom question the form asked.

`Needs Graph Fetch?` lets a flat test payload skip the API entirely, so you can develop without
Meta credentials.

### 3. Phone normalisation (`Normalize Phone`)
Handles `+91 98765 43210`, `0091...`, `09876543210`, `9876543210` and produces E.164.

Numbers that already carry a country code are **left alone** — `DEFAULT_COUNTRY_CODE` is only
applied to clearly local formats. Guessing at international numbers is how you message a
stranger. An unusable number stops the run rather than silently sending nowhere.

### 4. CRM (`Lookup Lead in CRM` → `Lead Found?`)
Google Sheets as the CRM, keyed on Meta's `lead_id`.

- **Found** → `Load Existing Lead` rehydrates the stored conversation so Claude continues where
  it left off. It is never stateless.
- **New** → `Create New Lead` appends the row, `Set New Lead Context` initialises state.

Both branches emit the **same shape**, so everything downstream sees one contract.

> The lookup node has `alwaysOutputData: true`. Without it, a zero-row result ends the branch
> silently in n8n and new leads are never created.

### 5. Idempotency (`Duplicate Event?`)
Meta redelivers events. Each row stores `last_event_id`; a repeat of the same event routes to
No-Op. The booking call also sends an `Idempotency-Key`. Result: no duplicate CRM rows, no
duplicate opening messages, no duplicate appointments.

### 6. The AI (`Prepare WhatsApp Context` → `Generate AI Reply` → `Parse AI Reply`)
`Claude` is attached as the `ai_languageModel` of a Basic LLM Chain. It receives the lead's
details, the qualification state, the appointment state and a trimmed transcript, and returns
**structured JSON**:

```json
{
  "reply": "the WhatsApp message to send",
  "intent": "appointment_request",
  "qualification_status": "qualified",
  "next_action": "check_availability",
  "selected_slot": { "date": "", "time": "" },
  "customer_data": { "name": "", "email": "", "property_location": "" }
}
```

`Parse AI Reply` never trusts that blindly:

- extracts JSON even if the model wraps it in prose or a code fence
- falls back to a safe `reply_only` turn if parsing fails
- **rejects a `book_appointment` for a slot that was never actually offered**, downgrading it
  to `check_availability`

`Generate AI Reply` has an error output wired to `AI Failure Fallback`, so a rate limit or a bad
key sends a neutral holding message and flags for human handover instead of leaving the lead on
read.

### 7. Appointments (`Check Appointment Availability` → `Book Appointment` → `Verify Booking`)
The AI has **no calendar** and is told so explicitly. Real slots come from your booking API,
get formatted into the message, and only a slot the lead was genuinely shown can be booked.

`Verify Booking` is the single gate that decides whether the lead is told "booked":

```js
const confirmed = status >= 200 && status < 300 && !!id;
```

Anything else becomes `booking_failed` and an honest message. **The bot never claims a booking
the API did not confirm.**

### 8. Delivery + CRM (`Send WhatsApp Reply` → `Record Delivery` → `Update CRM`)
WhatsApp Cloud API. The response is checked — a failed send is written to the CRM as failed.
The CRM never claims you messaged someone you did not. Conversation history is appended and
trimmed to the last 40 turns.

`Booking Confirmed?` then sends the confirmation message, but only on a real booking.

---

## Setup

### 1. Import
n8n → Workflows → **Import from File** → `workflow.json`

### 2. Credentials
| Node(s) | Credential |
|---|---|
| `Claude` | Anthropic API |
| `Lookup Lead in CRM`, `Create New Lead`, `Update CRM` | Google Sheets OAuth2 |

### 3. Environment variables
```bash
META_VERIFY_TOKEN=          # any random string you also paste into Meta
META_ACCESS_TOKEN=          # Meta Graph API token with leads_retrieval
WHATSAPP_ACCESS_TOKEN=      # WhatsApp Cloud API token
WHATSAPP_PHONE_NUMBER_ID=   # from Meta → WhatsApp → API Setup
BOOKING_API_URL=            # your calendar/booking service base URL
BOOKING_API_KEY=
CRM_SHEET_ID=               # the /d/<THIS>/edit part of the Sheet URL
DEFAULT_COUNTRY_CODE=91     # country the ads run in
MAX_HISTORY_TURNS=20        # optional, default 20
```

> If your n8n has `N8N_BLOCK_ENV_ACCESS_IN_NODE=true`, `$env` expressions resolve empty.
> Set it to `false` or move the values into node parameters.

### 4. Google Sheet
Create a sheet, name the tab **`Leads`**, and put these in row 1:

```
lead_id · name · phone · email · campaign_id · campaign_name · ad_id · ad_name
form_id · source · conversation_id · conversation_status · qualification_status
appointment_status · appointment_id · appointment_date · appointment_time
conversation_history · last_event_id · last_message_at · created_at · updated_at
```

Keep it **private**. It holds names, phone numbers, emails and addresses — connect n8n with
OAuth rather than sharing the link.

### 5. Booking API
`Check Appointment Availability` expects `GET {BOOKING_API_URL}/availability?from=&days=`
returning slots as `[{date, time, id}]`, `{slots:[...]}` or `{data:[...]}`.

`Book Appointment` posts to `{BOOKING_API_URL}/appointments` and must return a `2xx` with an
`id`, `appointment_id` or `booking_id`. Anything else is treated as a failure.

Swap these two nodes for a Google Calendar or Cal.com node if you prefer.

### 6. Point Meta at it
Meta App → Webhooks → Page → `leadgen`:
```
Callback URL:  https://<your-n8n>/webhook/meta-leads
Verify Token:  <META_VERIFY_TOKEN>
```

---

## Testing without Meta

`Manual Test Trigger` → **Execute Workflow**. `Test Meta Lead` injects:

```json
{
  "lead_id": "TEST-LEAD-001",
  "name": "Test User",
  "phone": "+923001234567",
  "email": "test@example.com",
  "campaign_name": "Test Campaign"
}
```

It bypasses the Graph API, so you can exercise phone normalisation, the CRM branch, Claude and
the routing logic with no Meta credentials at all.

---

## Design decisions worth knowing

**The AI writes messages. It does not perform operations.** Booking, CRM writes and WhatsApp
sends are all n8n nodes. The model's output is parsed, validated and constrained before any of
them run.

**Nothing is confirmed to the customer unless an API confirmed it.** Availability comes from the
booking service. Bookings are verified by status code plus a returned id.

**Failures are visible, not silent.** WhatsApp and booking calls use `neverError` + `fullResponse`
so the status code can be inspected and recorded rather than throwing mid-flow.

**Secrets stay out of the file.** Every credential is `$env` or an n8n credential reference.

---

## Known gaps

- **Inbound WhatsApp replies are not wired yet.** This workflow fires when a *new Meta lead*
  arrives. Handling the lead's *reply* needs a second webhook subscribed to WhatsApp's
  `messages` field, feeding `Extract Lead Event` with `inbound_message` set. All the state,
  history and qualification logic is already in place for it.
- **Appointment reminders** (24h / 2h before) are not implemented.
- **Rescheduling and cancellation** are not implemented — they route to human handover.

## License

MIT
