# Raid Game — Website (read-only) Implementation

The UI half of the kennyBot raid game. The full design and the backend live in
the **kennyBot** repo (`docs/raid-game-spec.md` and `docs/IMPLEMENTATION.md`);
this doc covers only what this site renders.

> **This repo is public** (it builds okrafans.com via GitHub Pages). Nothing in
> here may contain secrets, credentials, or operational/infrastructure detail.
> The Firebase Web `apiKey` is *not* a secret — it's designed to live in public
> client code (see the existing OKRAMARKET poll); access is controlled by
> Firebase **security rules**, not by hiding the key.

---

## 1. Role of the website: a read-only viewer of game state

The site is the **read layer**. All authoritative game state is written by the
bot via the Firebase Admin SDK; the website only **reads and renders** it. This
mirrors the OKRAMARKET poll, with one deliberate difference:

- The poll **writes** vote counts from the client (anonymous `runTransaction`),
  which is why its rules are wide open.
- **Game state must be client-read-only.** The site must never write to
  `players`, `bosses`, `raids`, `items`, `leaderboard`, or `config`. The backend
  locks those paths to `".write": false`; client-side write attempts will (and
  should) fail. Don't design any feature that needs a direct client write — if a
  viewer action is ever needed (e.g. "equip from the site"), it goes through a
  chat command or a validated Cloud Function, never a direct RTDB write.

Treat the site as a dashboard over data you don't own.

---

## 2. What to render — BUILT so far

The combat model is **active/automated** (spec §5.8): a weekly *muster* phase
ends in a scheduled *raid night* whose battle plays out automatically. The UI is
two pages, both `noindex` + unlinked until launch:

**`/raid/` — the muster / staging page** (`_includes/raid.html`). Phase-aware on
`config/raid.phase` (`signup → locked → live → done`):
- **countdown** to raid night (`config/raid.startsAt`),
- **team stats** tiles (power / defense / healing / hero count),
- **role-readiness meters** (aggregate role rating vs. boss thresholds, with the
  "need more healers!" social nudge),
- **the roster** — each signee with class/role/level + rarity-colored equipped
  gear,
- on `live`/`done`: a CTA to the battle/replay; on `done`: the result.
- Plus the **hero lookup card** (by Twitch name or `?u=`).

**`/live/` — the live battle** (`_includes/live.html`). A **replay player**:
boss HP bar, party grid (per-hero HP bars, dimmed when down), turn counter, and a
scrolling **combat log** revealed on a timer (see §8). Renders real `combat/log`
data when present; otherwise plays a seeded **demo battle** (`genDemoBattle()`)
so the page is demonstrable now. A "replay from start" control re-watches.

Both degrade to a labeled **PREVIEW** demo when Firebase is unreachable/empty, so
local preview works even offline.

**`/items/` — the Item Compendium** (`_includes/items.html`). The whole gear
catalog, filterable by role / slot / rarity / set, sortable, with up to four
items pinned for side-by-side comparison. Reads Firebase `items/` when it can and
falls back to the build-time snapshot in `_data/items.json`.

**Still to build:** per-hero inventory detail view, season banner.

### 2a. The gear contract (keep in step with kennyBot)

Two facts the UI must state, because getting them wrong is what confuses players:

1. **Gear serves exactly one role, and only that role's classes can equip it.**
   An item's rating comes from `bonuses[wearer.role]`, so off-role gear is worth
   zero — kennyBot now refuses the equip outright. The Compendium therefore
   carries a **"Wearable by"** column derived from the item's `role`
   (tank→Guardian · healer→Mender · dps→Berserker/Arcanist/Ranger). That mapping
   mirrors `CLASSES` in kennyBot's `src/content/classes.js`; if a class is ever
   added there, update `CLASSES_BY_ROLE` in `_includes/items.html` to match.
2. **Gear arrives by two different routes**, and they behave differently:
   *chat drops* are an open 60s lottery (`!grab` enters, one random winner), while
   *raid rewards* are automatic — clearing the weekly boss pays every hero on the
   roster one item in their own role, with surviving and MVP raising the rarity
   floor. `/how-to-play/` is the canonical player-facing explanation of both;
   don't restate the rules elsewhere, link to it.

Two more contracts the UI must not drift from:

3. **Enlistment is SEASON-LONG.** `!muster` once and the hero is on the roster
   for every remaining week — `setupRaidWeek` carries signups forward within a
   season; only week 1 of a new season starts empty. Never phrase muster as a
   weekly signup anywhere on the site.
4. **Leaderboards are PER ROLE.** `leaderboard/<seasonId>/<uid>` now holds
   `{ damage, healing, taken, role, raids }`. Each role is ranked on its own
   metric (dps→damage, healer→healing, tank→taken), mirroring `ROLE_METRIC` in
   kennyBot's `src/db/leaderboard.js`. Entries with no `role` are from seasons
   recorded before this and fall back to the old combined damage board.
   `taken` deliberately excludes AoE, which hits everyone equally and so ranks
   nobody.

The Compendium also shows a **Set tier** column: filling all three slots with
usable gear grants a % of gear rating, tiered by the *weakest* piece. That table
mirrors `config.rating.setBonusPct` — keep `SET_BONUS_PCT` in
`_includes/items.html` in step with it.

**The catalog lives in ONE place.** `src/content/items.js` in kennyBot is the
source of truth; `seedCatalog()` publishes it to Firebase `items/` on every boot
(idempotent, and it prunes ids dropped from the catalog). The Compendium reads
that node and nothing else — the same way `/raid/`, `/leaderboard/` and
`/arena/` read their nodes.

There used to be a second, build-time copy in `_data/items.json`. It has been
removed. It bought nothing — the page cannot function without Firebase anyway —
and it created a failure mode with no symptom: deploying the site shipped a
NEWER catalog than the database it renders, so the page quietly showed the older
one. That is exactly what happened when the 699-item expansion went out ahead of
the bot, and the page sat at 72 items looking perfectly healthy.

Consequence to remember: **a catalog change is a BOT deploy, not a site deploy.**
Edit `items.js`, deploy kennyBot, and the page updates live through `onValue`
with no site rebuild. If the node is empty or unreachable the page says so
rather than rendering something else.
