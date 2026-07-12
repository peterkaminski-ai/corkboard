# Wishlist

Ideas we like but haven't built. Roughly ordered by how much they'd change daily use.

## 1. Serverless mode — let board.html write files itself (File System Access API)

Today `corkboard-serve` exists for exactly one reason: the board's contract is "plain files in the repo" — the page has to write `inbox.jsonl` and `board.json` where the desk agent and git can see them, and browser JS can't touch the local filesystem. The server's whole job is four file writes and a fresh file read.

The escape hatch is the [File System Access API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) (Chromium browsers only — Chrome, Edge, and kin; not Firefox, not really Safari). The page shows a one-time **Connect to folder** button; the principal points it at the corkboard directory; the page stores the directory handle in IndexedDB. From then on it reads `board.json` directly and writes notes itself — no process to start, the board works the moment you open the HTML file.

What it would take:

- **A file-system adapter behind the existing seams.** All server traffic already funnels through `post()` and two `fetch` calls in `board.html`. Swap in an adapter: `fetch('/board.json')` becomes a directory-handle read; each `POST` becomes a read-modify-write of the JSON/JSONL files. Roughly 150 lines.
- **Per-note inbox files instead of one appended `inbox.jsonl`** — the one real design change, and worth making even without serverless mode. The File System Access API has no true append: "append" means rewrite-the-whole-file, which opens a race — the principal queues a note at the same moment the desk agent drains and truncates, and the rewrite resurrects everything just archived. One file per note (e.g. `inbox/2026-07-11T2110-260711-ye.json`) has no rewrite and no race, and the agent's drain barely changes (glob instead of readlines).
- **`render` becomes optional in live mode** (the page always reads fresh) but stays for producing shareable static snapshots.
- **Fallbacks for non-Chromium browsers:** the existing read-only degradation stays as-is; a "download queued notes" button could cover write-side use without the API.

Open question to spike first: whether the browser grants persistent permission ("Allow on every visit") to a page opened via `file://`, or only to one served from `localhost` — that determines whether serverless mode is truly zero-friction after the first grant or costs one re-confirm click per browser session.

The server should stay in the repo either way: it's the portable path for Firefox/Safari, and `curl`-able endpoints are genuinely useful for agents testing against the board.
