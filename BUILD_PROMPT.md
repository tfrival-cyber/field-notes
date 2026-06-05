# build prompt

field notes was built conversationally with [Claude Code](https://claude.com/claude-code) —
it grew over a series of messages rather than one giant prompt. what follows is that
intent distilled into a **single prompt you can paste into Claude to reproduce the app
from scratch**, plus notes on what each part is doing and what needed iteration.

> this is a faithful reconstruction, not a transcript. the app is the source of truth;
> this prompt is the shortest path to rebuilding something equivalent.

---

## the prompt

paste everything in this block into Claude (Claude Code works best, since the app
shells out to the `claude` CLI):

```
Build me a small, local "moodboard" web app called Field Notes for designers.
I drop travel photos in; it names and tags each one and files it by trip. It should
run on my own machine with no accounts, no cloud upload, and no build step.

Hard constraints:
- Zero dependencies. One server.js using only Node's standard library (http, fs,
  path, child_process, crypto). No npm install, no node_modules, no framework.
- One index.html for the entire frontend in vanilla JS/CSS. No React, no bundler.
- macOS is fine to assume. Use the built-in `sips` for image conversion/resizing
  and `mdls` for reading EXIF.
- Persist everything to disk: full-res images in ./library/, and metadata in a
  human-readable ./library.json. No database, no localStorage.

Image analysis:
- Analyse each image by shelling out to my local `claude` CLI in headless mode
  (`claude -p ... --output-format json --allowedTools Read`) so it uses my Claude
  Code login and needs no API key. Fall back to the Anthropic API only if
  ANTHROPIC_API_KEY is set and the CLI isn't found.
- For each image, return a small JSON record: a 2–3 word evocative name, a
  one-sentence description in a designer's voice, a category, a mood, 4 accurate
  hex colors sampled from the image, and up to 2 simple one-word materials
  (wood, stone, glass, metal, brass, ceramic, terracotta, textile, leather...).
- Send only a downscaled ~1024px JPEG copy to Claude for analysis (cheaper/faster);
  keep the untouched original on disk for display.

Trips + dates:
- Read GPS + capture date from each photo's EXIF (via mdls). Reverse-geocode the
  GPS to a city name and file the photo into a trip folder named after that city
  (e.g. "Antwerp"). Cache geocoding lookups on disk.

The interface (index.html):
- A drop zone at the top: drag JPG/PNG/WEBP/HEIC in, or click to pick files.
  Show each new card immediately with a loading state, then fill in once analysed.
- A responsive grid of cards. Each card shows the image, its name, and its palette
  as small swatches. Click a card to open a larger modal with the description,
  materials, and editable fields.
- Search, plus filters by category/mood/material/color, and sort by newest / A–Z /
  location. A sidebar of trip folders; clicking one filters to that trip.
- Drag a card onto a trip folder to reassign it.

Zine maker:
- Let me select several cards and lay them out as a printable zine. Two formats:
  (1) a single-sheet 8-panel mini-zine that folds with one cut (PocketMod imposition),
  and (2) a saddle-stitch booklet. Include a cover with the trip name + dates.
  Print via the browser's print dialog to US Letter.

Aesthetic:
- Quiet, editorial, print-inspired. Serif display type (EB Garamond) + a clean sans
  (Inter) for labels. Lots of whitespace, hairline rules, lowercase UI labels.
  It should feel like a designer's notebook, not a SaaS dashboard.

Start with the server + drop zone + analysis + grid, get that working end to end,
then we'll iterate on trips, then the zine.
```

---

## what each part is doing

- **"zero dependencies / one server.js / one index.html"** — the spine of the whole
  thing. forcing no framework and no build step keeps it a single `node server.js`
  away from running, which is what makes it feel like a tool rather than a project.

- **the drop zone** — the only way photos get in. it reads files in the browser and
  POSTs the original bytes to the local server; all the heavy image work happens
  server-side so nothing depends on flaky browser image libraries.

- **the vision call** — the heart of it. the server asks the `claude` CLI to *look*
  at each photo and return structured JSON. the exact instruction it sends is worth
  copying verbatim if you rebuild — it's tuned to give short, evocative names and a
  small, consistent material vocabulary instead of rambling captions:

  ```
  You are helping an industrial designer build a moodboard from travel photographs.
  Look at the image and return ONLY a JSON object, no markdown, in this exact shape:
  {
    "name": "2-3 word evocative name",
    "description": "one short sentence about what's in the image, in a designer's voice",
    "category": "one of: object, material, space, type, texture, detail",
    "mood": "one of: bold, quiet, warm, sharp, textured, weird",
    "colors": ["#hex1", "#hex2", "#hex3", "#hex4"],
    "materials": ["material1", "material2"]
  }
  Be specific and evocative. Name the actual thing you see. Pull real, accurate hex
  colors sampled from the image. Keep the description to one sentence. For materials,
  use simple common one-word names (wood, stone, glass, metal, brass, ceramic,
  terracotta, textile, leather...). Give at most 2, and avoid compound names
  ("stone", not "weathered cobblestone").
  ```

- **the library grid** — the payoff. cards render from `library.json`, so the data
  model and the UI are the same thing; you can hand-edit the JSON and the grid
  updates. the modal makes name/description/trip editable.

- **trips + EXIF** — what makes it feel smart. because location and date come from
  the photo itself, the collection organises itself; you never tag a trip by hand.

- **persistence** — deliberately boring. files on disk, one readable JSON. it
  survives restarts, it's backup-able, it's inspectable, and there's no migration.

- **the zine** — the reason it's not just a viewer. the mini-zine imposition (which
  panel goes where so a folded sheet reads in order) is the fiddliest single piece;
  it's worth asking Claude to lay it out and then verifying with page numbers.

- **the aesthetic** — stated up front so it's baked in from the first render rather
  than bolted on. naming the actual typefaces and the "notebook, not dashboard"
  feeling does most of the work.

---

## what worked, what needed iteration

**worked first time**
- the zero-dependency Node server + vanilla frontend. no toolchain to fight.
- shelling out to the `claude` CLI for analysis — no key management, just works if
  you're logged into Claude Code.
- disk persistence. one readable JSON file removed a whole category of problems.

**needed iteration**
- **the material vocabulary.** early tags were over-specific and inconsistent
  ("weathered cobblestone", "stained beech"). constraining the prompt to a short
  list of simple one-word materials, max two, fixed it.
- **the zine imposition.** getting the fold order right (which panel, which rotation)
  took a couple of passes; adding optional corner page numbers to verify the fold
  was the thing that made it trustworthy.
- **printing.** the browser print dialog behaved differently across browsers; the
  reliable path was serving each built zine at a real URL and printing that, locked
  to US Letter.
- **performance.** the grid was loading full-res originals and crawled; the fix was
  generating downscaled copies on demand and caching them.
- **duplicate imports.** the drop handler could fire twice and a re-added photo would
  duplicate; fixed with a content-hash check server-side and stopping the double-fire
  in the browser.

**what i'd refine if i built it again**
- make it cross-platform instead of leaning on macOS `sips`/`mdls` (e.g. a small
  pure-JS fallback for resize + EXIF) so it isn't Mac-only.
- decouple the data model from filenames a little more, so moving the library around
  is friction-free.
- decide the zine imposition math once, up front, with a test — it's the one part
  where "looks right" and "is right" can diverge.
```
