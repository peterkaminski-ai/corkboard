# Corkboard procedures — drain and sweep

Operating procedures for the desk agent (see `PATTERN.md` for the design). If the estate has more than one desk agent, these are **desk-holder work**: whichever agent currently holds the desk drains and sweeps; the others read the board but don't write board state.

## Drain — consume the principal's feedback (session start + natural breaks)

The board's feedback notes queue in `inbox.jsonl`. Draining is a desk habit, same rhythm as reading the day's notes: do it at session start, and glance again at natural breaks.

1. **Read `inbox.jsonl`.** Empty → done. Each line is an event: `feedback` (the principal's free-text note on an item), `action` (a bulk verb applied via checkboxes — make current, mark done, archive, integrate, don't worry about this any more, background this, brief me on this), `theme` / `pin` (dropdown and pin changes the server already applied to `board.json` — records only), or `process` (the principal pressed the 🔔 Process-notes button: a board-wide "come drain me" flag with no item ID — its presence is the message).
2. **Act on each `feedback` and `action` event.** Look up the item by `id` in `board.json`. Unambiguous notes ("archive this", "retheme to House", "bump to waiting") and the standard action verbs just get done — edit the canonical status file, update the registry, run the errand. Some verbs have shapes: "integrate" on a `bg` item means the estate's normal background-run integration protocol, liveness checks included; "don't worry about this any more" means status `paused-hold` plus a closeout note, unless context clearly says archive; "background this" means pre-stage a full task brief for a background run and offer the principal the launch command; "brief me on this" means a written deep-dive on the item in the next briefing. Anything outbound, destructive, or authority-adjacent surfaces to the principal first, per whatever authority rules govern the desk — a board note is still input, and the normal gates apply to actions it triggers.
3. **Archive every consumed line.** Move it to `inbox-archive.jsonl` with two fields appended: `"disposition"` (one line: what was actually done) and `"acted"` (date). `theme` and `pin` events archive with disposition `"applied by server"` unless the change itself prompted follow-up work worth recording. Nothing is silently dropped — the archive is the audit trail.
4. **Truncate `inbox.jsonl`** to only the lines not yet consumed (normally: empty it).
5. **Reflect changes.** If acting changed registry fields (status, next, oneliner), update `board.json`, run `./corkboard-serve render`, and commit.

A note the desk agent can't act on yet (needs the principal, needs a live conversation) **stays in `inbox.jsonl`** — it keeps showing as "queued" on the board, which is honest. Don't archive a note just to clear the queue; an archived line with no real disposition is a silent drop wearing a paper trail.

## Sweep — refresh the board from ground truth (on demand)

A sweep is initiated by the principal or the desk agent — after a big work session, before a review, whenever the board feels stale. Not cron.

1. **Survey ground truth**: project status files (and their mtimes), background-run scratch directories and handoff notes, task lists, the agent's own memory. Two or three parallel survey passes (one over projects, one over background runs) work well when the estate is large.
2. **Merge into `board.json`:**
   - New items get IDs: `yymmdd-aa`, where `yymmdd` is the founding date best-effort (status file, first commit, memory; else the sweep date) and `aa` is two random lowercase letters, rerolled on collision. **IDs are immutable once assigned.**
   - Existing items: update `status` / `next` / `oneliner` / `last_swept`. Never reassign IDs; never delete items — archive is a status, not a removal.
   - Vanished directories: flag for the principal in the item's `flags` array; **don't auto-archive.** A missing dir may have been moved or renamed, and the human decides what its disappearance means.
3. **Note drift** — status-file claims vs. observable reality ("status file says shipped; the deploy log says otherwise") — in `flags`. Drift is a first-class display concern; the board shows it with a ⚠️ rather than papering over it.
4. **Update the top-level `swept` date**, run `./corkboard-serve render`, commit the registry and the rendered board together.
5. **Mention** notable changes (new items, status jumps, fresh drift) in the principal's next briefing.

## Serving

- `./corkboard-serve` → http://localhost:2675 (2675 = CORK on a phone keypad). Start it when wanted; it's fine for it not to be running — the rendered `board.html` degrades to read-only.
- `./corkboard-serve render` re-embeds `board.json` into `board.html`; run it after any registry edit so the file-opened board stays fresh.
