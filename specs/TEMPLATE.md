# <Area Name>

Status: draft
Type: area sub-workflow

One or two sentences: what this area does and when the router should pick it.

## Area identity

These become literal strings in the workflow, so write them as final text.

- `area_name`: `<snake_case_area>`
- `area_description`: `<one sentence — what this area handles, used by the Response Agent>`

## Input

What the Gateway passes in. Default is a single `user_message`; say so if you
need more.

| Field | Type | Notes |
|-------|------|-------|
| `user_message` | string | Raw Telegram message text |

## What it does

Numbered steps, in order. Plain language — describe behaviour, not nodes.

1.
2.
3.

## Extraction schema

Fill this in if an LLM pulls structured data out of the message. Delete the
section if it doesn't.

| Field | Type | Required | When missing |
|-------|------|----------|--------------|
| | | | |

Rules the extractor must follow (formats, time zone, what counts as
"unclear", what it must never invent):

-

## Outcomes

Every way this workflow can end. One row per exit path — this table is the
workflow.

| `status` | `success` | `details` says |
|----------|-----------|----------------|
| | | |

<!-- Example, from Google Calendar Area:
| `created`      | true  | What was added, with time range and link     |
| `slot_busy`    | false | The requested range overlaps an existing event |
| `missing_info` | false | Which detail was missing or unclear          |
-->

## Integrations

| Service | Credential type | Used for |
|---------|-----------------|----------|
| | | |

New credential, or one already in `docs/architecture.md`?

## Gateway changes

Adding an area means editing `JARVIS - Gateway` too. Confirm or adjust:

- Router Agent system prompt: add `<area_name>` to the **Areas** list
- Structured Output Parser: add `<area_name>` to the `selected_area` enum
- Switch: new branch matching `<area_name>`, wired to this sub-workflow

## Edge cases

Things that must not happen, and what to do instead.

-

## Open questions

Park anything undecided here. I'll ask about these before building rather
than guessing.

-
