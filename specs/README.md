# Specs

Descriptions of workflows that don't exist yet. `docs/` describes what JARVIS
*is*; `specs/` describes what it's going to be.

## The loop

1. Copy `TEMPLATE.md` to `specs/<area-name>.md`.
2. Iterate on it until it feels right. Half-finished is fine — the
   **Open questions** section exists so you can park what you haven't decided.
3. Hand it over: *"build specs/email-area.md"*.
4. I read the spec, flag anything that conflicts with the area contract or is
   still undecided, then build it in n8n and back it up here.
5. Move the spec to `specs/built/` once it ships.

## Why the template asks what it asks

Every area sub-workflow in JARVIS returns the same five fields
(`area_name`, `area_description`, `success`, `status`, `details`) so the
Gateway can treat areas interchangeably. See `docs/architecture.md`.

That contract decides most of the build. The two sections worth spending
your iteration time on:

- **Outcomes** — every way the workflow can end, each with a `status` code
  and the `details` text. This is the whole workflow in one table; the node
  layout mostly falls out of it. `Google Calendar Area` has exactly three:
  `created`, `slot_busy`, `missing_info`.
- **Extraction schema** — if an LLM pulls structured data out of the message,
  the exact fields, which are required, and what to default when missing.

Everything else I can infer or ask about. Those two I can't guess.

## Status

Each spec carries `Status: draft | ready | built` at the top. `ready` means
you think it's complete enough to build from — it's the signal I should stop
asking and start building.
