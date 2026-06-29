# OKRAFANS 🌱

Nikki BreAnne's campy, okra-powered site — built with **Jekyll** and hosted on **GitHub Pages**.

## Add a new meme

1. Drop the image into `assets/memes/`.
2. Add an entry to `_data/memes.yml` (newest dates show first):

   ```yaml
   - image: assets/memes/my-new-meme.jpg
     caption: a short campy caption
     stream: which stream it's from
     date: 2026-06-14          # YYYY-MM-DD
   ```
3. Commit & push. GitHub Pages rebuilds automatically.

## Project layout

| Path | What it is |
|------|------------|
| `_layouts/default.html` | The campy shell (header + footer) every page uses |
| `_includes/poll.html` | The OKRAMARKET prediction poll (Firebase-backed) |
| `raid.html` + `_includes/raid.html` | Raid **muster** page `/raid/` — countdown, roster, team stats (WIP, unlisted) |
| `live.html` + `_includes/live.html` | **Live battle** page `/live/` — automated combat-log replay (WIP, unlisted) |
| `_data/memes.yml` | The meme wall content |
| `assets/css/style.css` | All the styling |
| `assets/memes/` | New meme images go here |
| `docs/` | Design / implementation docs (e.g. `raid-game-ui.md`); excluded from the built site |
| root `*.gif` / `*.jpg` | Legacy images, still referenced by the meme wall |

## The poll

The OKRAMARKET poll stores votes in a Firebase Realtime Database. To run a
**new** question, change `POLL_ID` in `_includes/poll.html` to a fresh name
(e.g. `"requiem"` → `"next-game"`) and update the `.pollQuestion` text. The
database security rules allow any `polls/<id>` path, so new polls start at 0/0
automatically.

## Raid game (work in progress)

A chat-driven community raid game (powered by the **kennyBot** Twitch bot +
Firebase) is being built on the `/raid/` (muster) and `/live/` (automated battle)
pages — currently `noindex` and unlinked. The website is the **read-only** view
of game state. See `docs/raid-game-ui.md`.

## Local preview

```sh
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000>.
