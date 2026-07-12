# The Corkboard pattern

Corkboard is a working pattern for a household or estate run by a human principal and one or more AI desk agents: every project, background agent run, and standing thread pinned to one surface, each with a stable tracking ID, a theme, a status, and a **transient feedback slot** where the principal can leave a note that the desk agent parses, acts on, and takes down.

This document describes the pattern. `PROCEDURES.md` describes the two operating procedures (drain and sweep) that make it run. The software in this repo — a single HTML page and a tiny stdlib Python server — is a reference implementation, deliberately small enough to read in one sitting.

## The board is a view, not a source of truth

This is the load-bearing principle. Canonical state lives where the work lives: project status files, background-run handoff notes, task lists, the estate's own repos. The board *reflects* that state as of the last sweep, and *routes* the principal's reactions back into it. It never becomes the place where truth is edited directly.

The transmission between the two is the desk agent. Feedback left on the board does not auto-sync into canonical files; the agent reads each note, acts on it (edits the status file, runs the errand, updates the registry), and records what it did. This is deliberate: a human-legible actor stays in the loop, every change has a narrator, and the audit trail reads like a person's work log rather than a sync daemon's diff.

## Non-goals

- **Not real-time.** The board updates when a sweep runs — initiated by the principal or the desk agent, whenever appropriate. Staleness is expected and displayed ("swept 3 days ago"), never hidden.
- **Not a task manager.** Task lists and per-project status files keep their jobs. The board is the surface where everything is visible at once, not the place where any one thing is managed.
- **Not cloud, not multi-user.** Localhost only, no auth, no accounts. The registry format doesn't preclude a hosted tier later, but the pattern starts at one desk.
- **Not auto-synced.** See above — the desk agent is the transmission, on purpose.

## Tracking IDs

Every item on the board gets a stable ID:

- **Format:** `yymmdd-aa`. The `yymmdd` part is the item's founding date when known (from its status file, first commit, or memory); when inscrutable, the date of the sweep that first saw it. Close is good enough — the date is a mnemonic, not a claim that must be accurate. The `aa` part is two random lowercase letters, rerolled on collision.
- **Assigned once, at first sweep. Immutable thereafter.** The ID survives renames, theme changes, status changes, even archival. It's the item's handle in feedback notes, conversation, and grep.
- Everything gets one: projects, background agent runs, and standing threads (an owed email, an expiring shared document, a key rotation) that the principal chooses to pin to the surface.

Items are never deleted, only given `status: archived` — the ID namespace stays whole, and old IDs in the archive still resolve.

## Kinds, themes, statuses

**Kinds.** Three: `project` (a directory of work with its own canonical status), `bg` (a background agent run with a scratch directory and a handoff), and `thread` (a standing obligation that isn't a directory — owed sends, expiring links, deadline-bound errands).

**Themes.** A small set of buckets, managed in the registry itself — adding, renaming, or retiring a theme is an edit there, made by the principal directly, by a feedback note, or by the desk agent. Each item holds at most one theme; `theme: null` is allowed, and the board shows an "Unthemed" bin so nothing hides.

**Statuses.** Six: `active` (being worked), `waiting` (blocked on the principal — a decision, a signature, a send), `steady` (healthy background hum, no action needed), `paused-hold` (deliberately parked), `done`, and `archived`. The board's default top section is the **attention queue** — everything `waiting` — because the single most valuable thing a board like this can answer is "what is blocked on me?"

## The data model — three files

All plain text, all git-versioned, all readable in any editor.

### `board.json` — the registry

Source of truth for the *board* (not for the estate). Themes, a top-level `swept` date, and an `items` map keyed by tracking ID:

```json
{
  "themes": ["Field Guide", "Print Shop", "Canoe"],
  "swept": "2026-07-10",
  "items": {
    "260415-km": {
      "title": "field-guide",
      "kind": "project",
      "path": "projects/field-guide/",
      "owner": "alice",
      "theme": "Field Guide",
      "status": "active",
      "oneliner": "Field guide to urban birds; 14 of 40 species accounts drafted",
      "next": "Corvid section this week; swift account blocked on the roost photos",
      "founded": "2026-04-15",
      "first_swept": "2026-06-02",
      "last_swept": "2026-07-10"
    }
  }
}
```

Optional per-item fields: `pinned: true` surfaces the item as a chip in the board's top strip; `flags` is an array of drift warnings (see the sweep, below) displayed inline with a ⚠️.

The `owner` field names who carries the item — the principal, a specific desk agent, or blank for unassigned.

### `inbox.jsonl` — transient feedback, one event per line

```json
{"ts": "2026-07-10T21:14:37-07:00", "id": "260519-mm", "type": "feedback", "text": "leaning toward the middle quote — anything in the fine print?"}
{"ts": "2026-07-10T21:16:05-07:00", "id": "260624-pn", "type": "action", "action": "archive"}
```

Five event types:

- `feedback` — a free-text note the principal typed on an item. Queued for the desk agent.
- `action` — a bulk verb the principal applied via checkboxes (make current, mark done, archive, integrate, don't worry about this any more, background this, brief me on this). Queued exactly like feedback.
- `theme` — a theme change from the item's dropdown. Deterministic, so the server applies it to `board.json` immediately **and** logs the event for the record.
- `pin` — a pin/unpin toggle. Also applied immediately and logged.
- `process` — the principal pressed the Process-notes button: a board-wide "come drain me" flag with no item ID. Its presence is the message.

**Lifecycle of a note:** the principal types it → it shows on the item as "queued" → the desk agent drains the inbox (parses, acts, updates canonical files and the registry) → the consumed line moves to `inbox-archive.jsonl` with a one-line disposition → the note disappears from the board. Nothing is silently dropped.

### `inbox-archive.jsonl` — the audit trail

Same shape as inbox events, plus two fields the agent appends when it consumes each line:

```json
{"ts": "2026-07-08T07:57:40-07:00", "id": "260531-cj", "type": "action", "action": "don't worry about this any more", "disposition": "status → paused-hold with a September re-ask note; both quotes kept in the project dir", "acted": "2026-07-08"}
```

The `disposition` is the point of the whole pattern: one line, in the agent's own words, saying what was actually done about the note. Read the archive top to bottom and you have a narrated history of every instruction the principal ever left on the board and its outcome.

## The sweep

A sweep refreshes the board from ground truth. It is desk-agent work, run on demand — after a big work session, before a review, whenever the board feels stale — not on a timer.

1. Survey ground truth: status files, background-run scratch directories, memory, task lists.
2. Merge into the registry: new items get IDs (founding date best-effort); existing items get `status` / `next` / `oneliner` / `last_swept` updates. IDs are never reassigned; items are never deleted.
3. Note drift — where a status file's claims disagree with observable reality — in the item's `flags` array. Drift is a first-class display concern, not something to quietly fix and hide.
4. Vanished directories get flagged for the principal, not auto-archived. A directory that disappeared might have been moved, renamed, or eaten by a mistake; the human decides.
5. Update the top-level `swept` date, re-render the static board, commit.

## The server and read-only degradation

`corkboard-serve` is a single-file Python 3 stdlib server, localhost only, no auth, no daemon ambitions — start it when wanted, and it's fine for it not to be running. It serves the board UI and appends events to the inbox; deterministic changes (theme, pin) it applies to the registry directly.

The board must also work with **no server at all**: `board.html` carries an embedded copy of the registry, so opening the file directly shows the last-swept board with inputs disabled and a hint to start the server. `corkboard-serve render` re-embeds the registry after any edit. This matters more than it looks: the read-only board is the guarantee that the surface survives the software.

## Why a human-legible agent in the loop

The obvious "improvement" to this pattern is automation: watch the filesystem, sync statuses both ways, drop the agent from the loop. Resist it. The value of the board is not that it updates itself; it's that a single accountable narrator — the desk agent — stands between the principal's one-line instructions and the estate's canonical files, and leaves a disposition every time it acts. That's what makes the board trustworthy enough to actually route decisions through: every note is either still visibly queued, or archived with an explanation. The moment changes happen with no narrator, the archive stops being an audit trail and the principal goes back to checking everything by hand.
