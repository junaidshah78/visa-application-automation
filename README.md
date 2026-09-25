# Visa Application Automation

An n8n workflow that runs a client's visa application process over WhatsApp — from first contact to document collection to status updates — with no manual data entry.

## What it does

1. **Intake (WhatsApp → Sheets).** A prospect messages the business on WhatsApp. The **OpenWA** trigger picks it up, checks Google Sheets to see if they're a new or returning applicant, and replies accordingly.
2. **Document capture.** When the applicant sends a photo of their passport or supporting documents, the workflow pulls the media from WhatsApp and uploads it to **LlamaParse**, polling until parsing finishes.
3. **Extraction & validation.** LlamaParse extracts structured fields (name, passport number, etc.) from the document. An **AI Agent** (OpenAI, with a structured output parser) validates and normalizes that data before it's written to the sheet.
4. **Status tracking.** Google Sheets acts as the applicant database, moving each row through states such as `Awaiting Documents` → `Pending` → `Approved`.
5. **Outbound updates.** A separate Sheets-triggered branch lets staff push a status change straight back to the applicant as a WhatsApp message — phone numbers are sanitized and validated first, and a bad row is skipped rather than stalling the whole batch.

## Stack

| Component | Role |
|---|---|
| [n8n](https://n8n.io) | Workflow orchestration |
| [OpenWA](https://github.com/rmyndharis/OpenWA) | Self-hosted WhatsApp API gateway (send/receive messages, media) |
| [LlamaParse](https://cloud.llamaindex.ai) | Document/image parsing and OCR |
| OpenAI (via LangChain nodes) | Structuring and validating extracted document data |
| Google Sheets | Applicant database, status tracker, and outbound-message trigger |

## Setup

1. Import `Visa_Application_Automation.json` into n8n.
2. Configure credentials:
   - **OpenWA API** — server URL + API key for your self-hosted OpenWA instance.
   - **LlamaParse API** — API key from LlamaCloud.
   - **OpenAI** — API key for the AI Agent / structured output parser.
   - **Google Sheets OAuth2** — access to the applicant tracking sheet.
3. Point the Google Sheets nodes at your sheet ID and confirm column headers match what the workflow expects (`Name`, `Number`, `State`, `Media_URL`, `Passport Number`, `Payment Status`, etc.).
4. Set the OpenWA **Session ID** in each OpenWA node to your active session's UUID (from Session → List All / Get Status).
5. Activate the workflow.

## Notes

- Uploaded media is currently proxied through a temporary file host before being handed to LlamaParse; swap this for persistent storage (Google Drive, S3) if the Media_URL needs to outlive the host's retention window.
- The lookup step that resolves a phone number to a WhatsApp chat ID (`Check Exists`) can return an `@lid` id rather than the raw number — downstream nodes should reference the resolved ID, not the phone number, when sending messages.
- Rows with an invalid or unresolvable phone number are flagged and skipped rather than failing the batch; check flagged rows periodically for data-quality issues in the source sheet.
