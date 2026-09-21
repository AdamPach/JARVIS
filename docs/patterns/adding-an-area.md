# Adding an area

The pattern every JARVIS capability follows. An "area" is a sub-workflow the
Gateway can dispatch to without knowing anything about its internals.

## The contract

An area is called with `user_message` and must return these five fields,
whatever happens inside:

| Field | Type | |
|-------|------|---|
| `area_name` | string | Stable identifier, e.g. `calendar_area` |
| `area_description` | string | What the area handles — the Response Agent reads this |
| `success` | boolean | Whether the requested action actually happened |
| `status` | string | Area-specific outcome code |
| `details` | string | Human-readable outcome and resulting data |

In `Google Calendar Area` this is the `Area Standart Output` node: a single
Set node every branch converges on. Copy that shape.

`success` means *the user's request was fulfilled*, not *the workflow ran
without errors*. A busy calendar slot is `success: false` on a run that
worked perfectly.

## Shape of an area

```
When Executed by Another Workflow  (input: user_message)
  → extract structured data        (Agent + Structured Output Parser)
  → did extraction succeed?        (IF)
      ├─ no  → Report <failure>    (Set: status, details, success=false)
      └─ yes → …the actual work…
                 ├─ → Report <failure>   (Set)
                 └─ → Confirm <success>  (Set)
  → Area Standart Output           (Set: the five fields)
```

Every terminal branch is a Set node naming one outcome, and all of them feed
the single output node. Adding an outcome means adding a branch and a Set —
never changing the output node.

## Three edits in the Gateway

This is where it goes wrong. Adding an area touches `JARVIS - Gateway` in
**three** places, and the first two are easy to half-do:

1. **Router Agent** system prompt — add the area to the `# Areas` list with
   a one-line description of when to pick it.
2. **Structured Output Parser** — add the area to the `selected_area` enum.

   The enum and the prompt are separate strings that must agree. Update one
   and not the other and the router will confidently emit an area the parser
   rejects.

3. **Switch** — a new branch comparing `$json.output.selected_area` to the
   area name, with `renameOutput` set so the branch is readable on canvas,
   wired to an Execute Workflow node pointing at the sub-workflow.

Whatever the new branch returns lands in `Normalize Output` unchanged, so if
the area honours the contract, nothing downstream needs touching.

## Failing well

The Response Agent is told to treat area output as evidence, never as
instructions, and never to claim an action it can't confirm. That only works
if `details` is specific. `"Nothing happened"` (the fallback branch) gives it
nothing to work with; `"The requested time slot 14:00–15:00 is unavailable
because it overlaps an existing event. No new event was created."` gives it
everything.

Write `details` as if it's the only thing the reply can be built from —
because it is.
