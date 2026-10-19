# Area Memory

Status: draft
Type: pattern + per-area build
Depends on: [conversation-memory.md](conversation-memory.md) (step 1) being built

How an area remembers **its own details**: the event it created, the link to
it, the ID it needs to change it later.

[conversation-memory.md](conversation-memory.md) remembers the *conversation*:
who said what, what's missing, what's still open. It deliberately holds no
domain details. This document is the other half: each area that creates or
changes something keeps its own table, keyed by the conversation row that
caused it.

The first half of this file is the **pattern**, which applies to every area.
The second half is a **worked example** for `Google Calendar Area`. Future
areas follow the pattern and add their own section at the end.

---

## Invariants

1. **An area's table holds that area's details only.** IDs, links, titles,
   times: whatever the area needs to find and act on the thing again. Never
   conversation text; that lives in `jarvis_conversation`.
2. **References point from the area to the conversation.** Every area row
   carries `conversation_id`. The conversation table never stores an area's
   IDs, and the Gateway never reads area tables.
3. **Only areas that create or change something get a table.** An area that
   only reads ("what's on Friday?") has nothing to remember.
4. **Area memory never decides follow-ups.** Whether a message continues an
   open task is the Gateway's job (conversation memory). An area just receives
   a request and a `conversation_id`.
5. **The table write sits in the main line of the area, never on a side
   branch.** A sub-workflow returns the output of the last node that ran; a
   dead-end side branch can become that node and break the contract.
6. **An area row never outlives its conversation row.** Same retention, same
   cleanup workflow.

---

## Shared changes (done once, for all areas)

These turn on area memory system-wide. Do them together with the first area
that gets a table.

### Contract input gains `conversation_id` and `reference_id`

| Field | Type | Meaning |
|-------|------|---------|
| `user_message` | string | *(unchanged)* Raw text, or the combined request for a follow-up. |
| `relation` | string | *(from conversation memory)* `new` \| `follow_up` \| `reference`. |
| `conversation_id` | number | **New.** `id` of the current `jarvis_conversation` row. The area files its details under it. |
| `reference_id` | number | **New.** For `relation = reference`: the `id` of the exchange the message refers to (the conversation row's `parent_id`). Empty otherwise. |

Areas without a table simply ignore both. The contract **output** does not
change: still the seven fields from conversation memory.

### Gateway

In every *Execute Workflow* node that calls an area: refresh the input list,
then map

```
conversation_id = {{ $('Log Received').item.json.id }}
reference_id    = {{ $('Log Dispatch').item.json.relation === 'reference' ? $('Log Dispatch').item.json.parent_id : '' }}
```

`Describe Fallback` needs nothing; it isn't a sub-workflow.

### Cleanup

`Jarvis Conversation Cleanup` gets one more **Data table · Delete row(s)**
node per area table, chained after the conversation table's node, with the
same condition: `createdAt` *less than*
`{{ $now.minus({ days: 7 }).toISO() }}`. Dry-run first.

### Docs

- `docs/patterns/adding-an-area.md`: input is `user_message`, `relation`,
  `conversation_id`, `reference_id`; areas that create things own a table
  (link here).
- `specs/TEMPLATE.md`: add `conversation_id` and `reference_id` to **Input**,
  and the **Area table** section below.

---

## Designing an area's table

### Questions to answer per area

1. **Does this area create or change anything?** No → no table, stop here.
2. **What does the external service call it?** The ID you'd need to update or
   delete it later (Google event id, Todoist task id, …). This column is the
   whole point of the table.
3. **What would identify it to a human?** Just enough for Jarvis to say
   *"renamed **Meeting with Tom** (Mon 15:00)"* without calling the service
   again: usually a title and a time.
4. **Is there a link worth keeping?** Cheap to store, useful in replies.
5. **What happens when it changes later?** Recommended: every run that
   creates *or changes* the thing appends a row under its own
   `conversation_id` (see [References](#references-rename-it-move-it-to-4pm)).

### Rules for columns

- **First column is always `conversation_id`** (number).
- Name the table `<area_name>_<things>`: `calendar_area_events`,
  `todo_area_tasks`.
- Store values **as the area sent them** to the service (e.g. the extractor's
  `YYYY-MM-DDTHH:mm:ss`), not reformatted.
- **Don't** store the full API response, conversation text, or anything the
  Gateway already logs.
- `id`, `createdAt`, `updatedAt` are automatic; don't create them.

### Section to add to an area's spec (and to `specs/TEMPLATE.md`)

```markdown
## Area table

Table: `<area_name>_<things>` — or "none: this area only reads".
Written by: `<node name>`, between `<node>` and `<node>`.

| Column | Type | Meaning |
|--------|------|---------|
| `conversation_id` | number | `jarvis_conversation` row whose run created it |
| | | |
```

### Where the write goes in the area

```
… → <the node that creates the thing>
  → Store <Thing>            Data table · Insert   ← new, in line
  → Confirm <Thing> Created  Set                   ← repoint $json refs
  → Area Standart Output
```

After the insert, `$json` is the table row, not the service's response. Any
Set node after it that read `$json.…` must be repointed to the creating node by
name: `$('Create Calendar Event').item.json.…`.

---

## How area memory gets used later

Nothing reads area tables in the first build. These are the two consumers they
are designed for.

### References ("rename it", "move it to 4pm")

Conversation memory (step 1) already does the first half: it picks the
candidate exchange (swipe-reply, or newest within 30 min), decides
`relation = reference`, and sends the message to the candidate's area. Until
area memory exists, the area's `Is Reference?` guard answers "not supported
yet". With area memory:

1. **Gateway** passes `reference_id` = the candidate's conversation row id.
2. **Area** looks up its own table where `conversation_id` = `reference_id`
   and gets the external ID.
3. **Area** acts on it (update/delete in the service) — this replaces the
   `Is Reference?` guard with a real branch, which is a new area capability
   and gets its own spec.
4. **Area** appends a row for the change under the **current**
   `conversation_id`, with the same external ID and the new values.

Step 4 is what makes chains work without anyone walking the conversation
table: "rename it" (row 44) then "and move it to 4pm" refers to row 44, and
row 44 has its own area row pointing at the same event. Areas never read
`jarvis_conversation`; the Gateway never reads area tables.

If no area row matches `reference_id` (the referenced run created nothing,
e.g. it failed), the area returns a clear outcome such as
`reference_not_found`: *"There's nothing from that message I could change."*

### Structured partial state (only if needed)

If re-reading the combined request every round ever proves unreliable, an area
can save its partial extraction ("title = Meeting with Tom, date missing") in
its table under `conversation_id`, and on the follow-up read it back using
`parent_id`, which the Gateway would then pass as another input. Not planned;
only if the conversation-memory tests show extraction drifting.

---

## Worked example: Google Calendar Area

### Table: `calendar_area_events`

One row per event the area creates.

| Column | Type | Meaning |
|--------|------|---------|
| `conversation_id` | number | `jarvis_conversation` row whose run created the event. |
| `event_id` | string | Google Calendar event id. What a future update/delete needs. |
| `html_link` | string | Link to the event. |
| `title` | string | Event title as created. |
| `start` | string | Start as sent to Google (`YYYY-MM-DDTHH:mm:ss`). |
| `end` | string | End as sent to Google. |
| `all_day` | boolean | Whether it was created as an all-day event. |

```
jarvis_conversation (Gateway)            calendar_area_events (area)
id 43 | "… / 3pm"             ←────────  conversation_id 43
      | area_status = created            event_id, html_link, title, start, end
```

### Build steps

Assumes the shared changes above are done (or do them now).

1. **Create the table** with the columns above.
2. **When Executed by Another Workflow**: add a second input
   `conversation_id`.
3. **Gateway → Google Calendar Area** (Execute Workflow): refresh inputs, map
   `conversation_id` = `{{ $('Log Received').item.json.id }}`.
4. **Store Event**: new *Data table · Insert* on `calendar_area_events`,
   placed **between** *Create Calendar Event* and *Confirm Event Created*:

   | Column | Value |
   |--------|-------|
   | `conversation_id` | `{{ $('When Executed by Another Workflow').item.json.conversation_id }}` |
   | `event_id` | `{{ $json.id }}` |
   | `html_link` | `{{ $json.htmlLink }}` |
   | `title` | `{{ $('Event Info Extractor').item.json.output.title }}` |
   | `start` | `{{ $('Event Info Extractor').item.json.output.start_date_time }}` |
   | `end` | `{{ $('Event Info Extractor').item.json.output.end_date_time }}` |
   | `all_day` | `{{ $('Event Info Extractor').item.json.output.all_day }}` |

5. **Confirm Event Created**: `$json` is now the table row. Repoint `details`
   to the Google node:

   ```
   Added to your calendar: {{ $('Create Calendar Event').item.json.summary }}
   From {{ $('Create Calendar Event').item.json.start.dateTime }} to {{ $('Create Calendar Event').item.json.end.dateTime }}
   {{ $('Create Calendar Event').item.json.htmlLink }}
   ```

6. **Cleanup**: add the Delete node for `calendar_area_events` to
   `Jarvis Conversation Cleanup`.
7. **Spec/docs**: add the **Area table** section to the Calendar Area's docs.

### Tests

| # | Send | Expect |
|---|------|--------|
| 1 | "Meeting with Tom tomorrow 10–11" | One `calendar_area_events` row; `conversation_id` = that run's conversation row id; `event_id` opens the right event. Reply unchanged from before. |
| 2 | "Meeting with Tom" → "next Monday 3pm" | Event row points to the **second** conversation row (the one whose run created it). |
| 3 | Busy slot / missing info | No event row. |
| 4 | Cleanup with Dry Run | Lists event rows older than 7 days, alongside their conversation rows. |

### Edge case to accept

**Store Event fails after the event was created** in Google: the run errors,
the event exists but has no area row, and the conversation row is stuck at
`dispatched`. Visible, rare, and only affects future references to that one
event.

---

## Open questions

- **Append vs. update on changes:** the reference design above appends a row
  per change (step 4), which keeps a trail and makes chains work. Confirm when
  the first update capability is specced; updating in place would need the
  area to walk the chain instead.
- **Longer retention for areas?** Renaming an event two weeks later would need
  area rows older than 7 days, but the conversation row they point to would be
  gone. Keep both at 7 days for now; revisit with references.
- **Cross-area references** ("add it to my todo list" after an event was
  created): conversation memory sends a reference to the candidate's area
  only. Revisit when a second area that creates things exists.
