# Architecture

How JARVIS fits together end to end.

## Overview

Telegram is the only entry point. `JARVIS - Gateway` receives the message,
checks the sender, classifies intent, dispatches to an area sub-workflow,
then hands the result to a response agent that writes the reply.

```
Telegram Trigger
  → Filter (allowlist: only my Telegram user id passes)
  → JARVIS Router Agent   (Mistral) → picks calendar_area | fallback
  → Switch
      ├─ calendar_area → Google Calendar Area (sub-workflow)
      └─ fallback      → Describe Fallback
  → Normalize Output
  → JARVIS Response Agent (gpt-5.6-luna) → writes the reply
  → Send a text message
```

### The area contract

Every area sub-workflow returns the same five fields, so the Gateway can
treat areas interchangeably. `Google Calendar Area` emits them from its
`Area Standart Output` node; the fallback branch fakes the same shape.

| Field              | Meaning                                    |
|--------------------|--------------------------------------------|
| `area_name`        | Area identifier, e.g. `calendar_area`      |
| `area_description` | What the area handles                      |
| `success`          | Whether the requested action happened      |
| `status`           | Area-specific code, e.g. `slot_busy`       |
| `details`          | Human-readable outcome and resulting data  |

This is the main pattern in the system — adding an area means matching this
contract and adding a Switch branch. See `docs/patterns/`.

## Workflows

| Workflow | File | Purpose |
|----------|------|---------|
| JARVIS - Gateway | [jarvis-gateway.json](../workflows/jarvis-gateway.json) | Telegram entry point, routing, reply generation |
| Google Calendar Area | [google-calendar-area.json](../workflows/google-calendar-area.json) | Extracts event details, checks availability, creates the event |

Not yet backed up: `Jarvis` (older Telegram assistant, inactive),
`Jarvis Memory`, `Jarvis Memory Cleanup`.

## Integrations

What a fresh n8n instance needs before an import will run. Names and types
only — no keys, tokens, or secret values in this file.

| Integration | Credential type | Used by |
|-------------|-----------------|---------|
| Telegram | `telegramApi` | Gateway trigger + send |
| Mistral Cloud | `mistralCloudApi` | Router agent (`mistral-small-2603`) |
| OpenAI | `openAiApi` | Response agent + event extractor (`gpt-5.6-luna`) |
| Google Calendar | `googleCalendarOAuth2Api` | Availability check + event creation |

## Restoring a redacted export

Exports in `workflows/` are sanitized for publication. After importing,
put these back by hand:

| Placeholder | Where | What it was |
|-------------|-------|-------------|
| `REDACTED_TELEGRAM_USER_ID` | Gateway → `Filter` | My Telegram user id — the allowlist. Until restored, the Filter blocks everyone. |
| `REDACTED@group.calendar.google.com` | Calendar Area → `Check Slot Availability`, `Create Calendar Event` | The "JARVIS" Google Calendar id |
| `"id": "REDACTED"` under `credentials` | Both files | Re-pick each credential in the n8n UI |
| `"webhookId": "REDACTED"` | Gateway Telegram nodes | n8n regenerates these on import |

The sub-workflow reference in the Gateway (`PLa0XwO0oKPHDWAQ`) is kept
deliberately so the wiring is readable; re-point it after importing into a
different instance.

## Data

No data tables or vector stores in these two workflows. `Jarvis Memory`
uses a `jarvis_context` table — document that here when it gets backed up.
