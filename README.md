# Summit Roofing – AI Quote Generator

An n8n automation that turns a roof photo and job description into a priced, branded PDF quote. Requests arrive from the customer web form or from the inbox agent via a webhook. The business owner reviews and approves each quote before anything is sent.

> Summit Roofing is a fictional company used for this demo. The workflow fits any trade business that quotes from a fixed price list.

**Demo video:** coming soon

![Workflow overview](screenshots/02-workflow-overview.png)

---

## Part of the Summit Roofing AI back-office

This workflow is one part of a small set of AI tools for Summit Roofing's back office. The **[Summit inbox agent](https://github.com/Uzelac97/summit-inbox-agent)** handles incoming quote requests and sends them to this workflow's webhook. Requests from email and from the website then go through the same pipeline, and the owner reviews them in the same way.

---

## The problem

Small roofing and trade businesses lose time on quote requests:

- Each request means reading the description, looking at photos, working out quantities, looking up prices and typing a quote document.
- Customers wait days for an answer and often go with whoever replies first.
- Quotes look different every time, and there is no central record of what was sent.

## The solution

The customer fills in a short web form and uploads a roof photo, or the inbox agent forwards a request to the webhook. The workflow then:

1. **Normalizes the input.** Both entry points are converted into one shape: name, email, address, description, photo and source (`form` or `email`).
2. **Analyzes the job.** Claude reads the description, looks at the photo and lists the work items and quantities.
3. **Prices it from your own price list.** Prices come only from your Google Sheet. The AI never sets a price.
4. **Builds a branded PDF quote.**
5. **Asks the owner for approval by email.** The owner can approve, adjust quantities and add a note, or decline.
6. **Sends the final quote to the customer** and logs it in Google Sheets.

**What the owner gets**

- Quotes ready to review minutes after a request comes in, without typing anything.
- Full control: nothing reaches the customer without the owner's approval.
- A warning when the AI is unsure, so the owner knows which quotes need a closer look.
- A log of every quote, whether it was sent, changed or declined, and where the request came from.
- An email alert if anything fails, so no request is lost.

---

## Walkthrough

| | |
|---|---|
| **1. Customer request.** The customer enters name, email, address and job description, and uploads a roof photo.<br><br>![Customer request form](screenshots/01-customer-request-form.png) | **2. Owner review email.** Total, AI confidence, urgency, the reasoning for each item and any missing info. The draft PDF is attached.<br><br>![Owner review email](screenshots/03-owner-review-email.png) |
| **3. Approval form.** The owner approves, approves with changes (e.g. `gutter_clean=18, moss_removal=20`) or declines, and can add a note to the customer.<br><br>![Owner approval form](screenshots/04-owner-approval-form.png) | **4. Quote sent to the customer.** The revised total and the owner's note are included, with the PDF attached.<br><br>![Customer quote email](screenshots/05-customer-quote-email.png) |

**5. Final PDF quote.** Generated from HTML by Gotenberg, using the corrected quantities.

![Final quote PDF](screenshots/06-final-quote-pdf.png)

**6. Low-confidence flag.** Here the photo did not show a roof. The subject line is marked *URGENT* and *Needs review*, estimated quantities are labelled, and the missing information is listed.

![Low-confidence review email](screenshots/07-low-confidence-flag.png)

**7. Error alert.** If any step fails (here, the PDF service was unreachable), the owner gets an email with the failed step and a link to the run.

![Error alert email](screenshots/08-error-alert-email.png)

---

## Key features

- **Two entry points, one pipeline.** The customer form and the `POST /webhook/summit-quote` webhook both feed a *Normalize input* node. Everything downstream sees the same fields, and each request is tagged with its `source`.
- **AI vision with a fixed vocabulary.** Claude may only use the `item_id`s from the price list and must return strict JSON. Prices are never part of the AI output.
- **Pricing in code.** Line totals and the grand total are calculated in a Code node from the Google Sheet. Items not found in the price list are reported to the owner, not priced.
- **Review flags.** A quote is marked *Needs review* when confidence is below 0.6, a quantity is estimated, or an item is unknown. High-urgency jobs get an *URGENT* subject prefix. Estimated quantities are marked with `*` in the PDF.
- **Human in the loop.** The Gmail *send and wait* node collects the owner's decision through a form. It waits up to 7 days.
- **Corrections by the owner.** Corrections use the `item_id=qty` format. `0` removes an item, and any price-list item can be added. Invalid input stops the run and triggers the error alert.
- **Branded PDF.** HTML is rendered to PDF by Gotenberg (headless Chromium). Customer input is HTML-escaped.
- **Logging.** Every outcome is appended to `Quotes_Log` with status `Sent`, `Sent with changes` or `Declined`, and with its source (`form` or `email`).
- **Reliability.** The Claude and Gotenberg calls retry on failure. A separate error workflow emails the owner when a run fails.

---

## How it works

```mermaid
flowchart LR
    A[Customer web form<br>photo + description] --> N[Normalize input<br>one shape + source]
    W[Webhook POST /summit-quote<br>from inbox agent] --> N
    N --> B[Claude<br>analyze photo + text]
    B --> C[Parse JSON]
    C --> D[(Google Sheets<br>PriceList)]
    D --> E[Calculate prices<br>+ review flags]
    E --> F[Build quote HTML]
    F --> G[Gotenberg<br>HTML to PDF]
    G --> H[Email owner<br>draft + reasoning]
    H --> I{Owner approval<br>Gmail send and wait}

    I -- Approve --> J[Attach original PDF]
    I -- Approve with changes --> K[Apply corrections]
    K --> L[Build revised HTML]
    L --> M[Gotenberg<br>revised PDF]
    M --> O2[Attach revised PDF]
    J --> O[Send quote to customer]
    O2 --> O
    O --> P[(Quotes_Log<br>Sent / Sent with changes)]
    I -- Decline --> Q[(Quotes_Log<br>Declined)]

    subgraph Error workflow
        X[Error Trigger<br>any failed run] --> Y[Email owner<br>failed node + execution link]
    end
```

**Webhook input.** Send a `multipart/form-data` POST to `/webhook/summit-quote` with the header `X-Summit-Key`:

| Field | Type | Notes |
|---|---|---|
| `name` | text | Customer name |
| `email` | text | Customer email, used for the final quote |
| `address` | text | Optional |
| `job_description` | text | Same content as the form's *Job description* |
| `roof_photos` | file | Roof photo (`.jpg` or `.png`) |

## Tech stack

| Component | Role |
|---|---|
| **n8n** (self-hosted) | Workflow engine, customer form, webhook, approval form |
| **Claude API** (Anthropic) | Image + text analysis, structured JSON output |
| **Gotenberg 8** | HTML to PDF conversion |
| **Google Sheets** | Price list and quote log |
| **Gmail** | Owner notifications, approval request, customer email |
| **Summit inbox agent** (separate repo) | Sends emailed quote requests to the webhook |
| **Docker Compose** | Runs n8n and Gotenberg locally |

---

## Setup

**Requirements:** Docker, an Anthropic API key, and a Google account (Sheets and Gmail). The webhook entry point also needs the inbox agent, or any client that can send the header below.

1. **Start the services**
   ```bash
   cp .env.example .env        # optional: change the timezone
   docker compose up -d
   ```
   n8n runs at http://localhost:5678. n8n reaches Gotenberg at `http://gotenberg:3000` inside the Docker network.

2. **Create the Google Sheet** with two tabs, `PriceList` and `Quotes_Log` (see [structure below](#google-sheets-structure)). `Quotes_Log` needs a `source` column.

3. **Add credentials in n8n** (*Credentials > Add*):
   - Anthropic API
   - Google Sheets OAuth2
   - Gmail OAuth2
   - Header Auth (for the webhook, see step 5)

4. **Import the workflows** (*Workflows > Import from file*):
   - Import `workflows/error-alert.json` first, then `workflows/ai-quote-generator.json`.

5. **Create the Header Auth credential for the webhook**
   - In n8n, go to *Credentials > Add*, search for **Header Auth** and choose it.
   - **Name:** `X-Summit-Key`. **Value:** a long random secret, for example the output of `openssl rand -hex 32`.
   - Save it with a label such as `Summit Inbox Agent key`.
   - In the *Webhook* node, select this credential under *Credential for Header Auth*.
   - Put the same secret into the inbox agent's configuration. Keep it out of git.

   The webhook rejects any request that does not send this header with the right value.

6. **Configure the imported workflows**
   - Assign your credentials to each Claude, Google Sheets and Gmail node.
   - In the three Google Sheets nodes, select your sheet (replacing `YOUR_SHEET_ID`) and the right tab.
   - Replace `owner@example.com` with the owner's address in *Email owner: draft*, *Owner approval* and the error workflow's *Send a message* node.
   - In the quote workflow's *Settings*, set **Error workflow** to *Summit Roofing – Error alert*.

7. **Publish (activate)** both workflows, then test both entry points.
   - **Customer form:** open the production URL of the *On form submission* node and submit a test request.
   - **Webhook:**
     ```bash
     curl -X POST https://YOUR_N8N_URL/webhook/summit-quote \
       -H "X-Summit-Key: YOUR_SECRET" \
       -F name="Test Customer" \
       -F email="test@example.com" \
       -F address="1 Test Street" \
       -F job_description="Gutter cleaning, about 14 m" \
       -F roof_photos=@roof.jpg
     ```
     The owner receives a review email. Once the owner approves or declines, the `Quotes_Log` row is written with `source` set to `email`.

> **Note:** The approval button in the owner's email links back to your n8n instance. For the owner to use it from another device, n8n must be reachable at a public URL (set `WEBHOOK_URL`), not only `localhost`. The same applies to the webhook URL for the inbox agent.

---

## Google Sheets structure

**`PriceList`**: one row per service. `item_id` and `unit` must match the values in the Claude prompt.

| item_id | description | unit | price_eur |
|---|---|---|---|
| gutter_clean | Gutter cleaning | m | 6 |
| tile_repair | Tile repair (single tiles) | piece | 12 |
| moss_removal | Moss removal and roof cleaning | m2 | 8 |
| inspection | On-site inspection | job | 90 |
| leak_fix | Leak detection and repair | job | 280 |
| … | | | |

All supported `item_id`s and their units:
`tile_replace` (m2), `tile_repair` (piece), `gutter_replace` (m), `gutter_clean` (m), `flashing_repair` (m), `ridge_repair` (m), `leak_fix` (job), `insulation` (m2), `skylight_install` (piece), `scaffolding` (job), `inspection` (job), `disposal` (job), `moss_removal` (m2).

**`Quotes_Log`**: the workflow appends one row per outcome.

| date | customer_name | email | job_summary | total_eur | status | source |
|---|---|---|---|---|---|---|
| 2026-10-05 15:13 | Sarah Mitchell | customer@example.com | Gutter cleaning and cracked tile repair | 340 | Sent with changes | form |

`source` is `form` for requests from the customer web form and `email` for requests received through the webhook.

---

## Repository structure

```
.
├── workflows/
│   ├── ai-quote-generator.json   # main workflow (20 nodes)
│   └── error-alert.json          # error workflow
├── screenshots/                  # README images, in walkthrough order
├── docker-compose.yml            # n8n + Gotenberg
├── .env.example
└── README.md
```

## Notes and limitations

- Quotes are **estimates**. The PDF states that the final price is confirmed after an on-site inspection.
- The AI's quantities depend on photo quality and on how detailed the description is. The review flags exist for exactly this reason, and the owner's approval is always required.
- Credential IDs, webhook IDs, the Google Sheet ID, the n8n instance ID and personal email addresses have been removed from the exported workflows. Placeholders: `YOUR_SHEET_ID`, `YOUR_WEBHOOK_ID_<n>`, `owner@example.com`, `customer@example.com`.
