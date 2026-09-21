# JARVIS

Personal AI assistant built in n8n. This repo is documentation and backup —
the assistant itself runs on my n8n instance.

## Layout

| Path         | What's in it                                          |
|--------------|-------------------------------------------------------|
| `workflows/` | Sanitized workflow exports. `raw/` is local-only.     |
| `prompts/`   | System prompts from AI Agent nodes, as plain markdown |
| `docs/`      | Architecture, per-workflow notes, reusable patterns   |
| `scripts/`   | Export and sanitization helpers                       |

## Backing up a workflow

1. In n8n: workflow menu → Download. Save into `workflows/raw/`.
2. Sanitize it into `workflows/` (see Sanitization below).
3. If the system prompt changed, update the matching file in `prompts/`.

Prompt files are named `<workflow>--<node>.md`, so the prompt for the agent
node in `jarvis-main.json` lives at `prompts/jarvis-main--agent.md`.

## Restoring

In n8n: Workflows → Import from File. Credentials are not included in
exports — reconnect them by hand after importing. The integrations JARVIS
needs are listed in `docs/architecture.md`.

## Sanitization

Exports are public-safe only after removing: `meta.instanceId`,
credential ids and names, `webhookId` and webhook paths, and any
hardcoded chat ids, emails, or phone numbers.

Never commit anything from `workflows/raw/`.
