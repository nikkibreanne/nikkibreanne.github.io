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

`_data/items.json` is a **generated** file — never hand-edit it. Regenerate from
kennyBot after any catalog change:

```
node scripts/export-catalog.mjs > ../nikkibreanne.github.io/_data/items.json
```

Compose as `_includes/*.html` into pages, the same way `index.html` composes
`poll.html` etc. The "take me home" banner now lives beneath the meme wall on the
home page (moved out of the footer).

---

## 3. Firebase client pattern (reuse what the poll proved)

Same SDK, same project config, **read-only** calls. Subscribe with `onValue` for
live updates; no `runTransaction`/`set`/`update` anywhere in game code.

```js
// reuse the public okrafans firebaseConfig already in _includes/poll.html
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
import { getDatabase, ref, onValue }
  from "https://www.gstatic.com/firebasejs/10.12.2/firebase-database.js";

const db = getDatabase(initializeApp(firebaseConfig));

// config/raid is the active-raid pointer: { seasonId, weekId, phase, locksAt, startsAt }
onValue(ref(db, "config/raid"), (snap) => {
  const cfg = snap.val();
  if (!cfg) return renderNoRaid();
  const key = `${cfg.seasonId}/${cfg.weekId}`;
  onValue(ref(db, `bosses/${key}`), (s) => renderBoss(s.val()));  // name, hp, thresholds, affix
  onValue(ref(db, `raids/${key}`),  (s) => renderRaid(s.val()));  // signups, team, combat
});
```

Reads map onto the spec §9 model + the combat extensions (`IMPLEMENTATION.md §L`):
`config/raid`, `bosses/<seasonId>/<weekId>`,
`raids/<seasonId>/<weekId>/{signups,team,combat}`, `players/<twitchUserId>`,
`usernames/<login>`, `leaderboard/<seasonId>`.

**Notes**
- Factor the shared `firebaseConfig` so it isn't duplicated between the poll and
  the game (e.g. one small `assets/js/firebase-config.js`, or a Jekyll include).
- Render defensively: every node can be missing/null (no season yet, boss not
  set, a player who hasn't played). Show friendly empty states, never crash.
- **Detach listeners** (`off()` / the unsubscribe returned by `onValue`) when a
  view is torn down, so SPA-style navigation doesn't leak connections (each open
  listener counts against the RTDB connection budget — see §6).
- Don't trust client time for the countdown; compute from the server-provided
  `config/raid.startsAt` (epoch ms) and accept minor skew.

---

## 4. Bars (extend the poll's bar idiom) — as built

The poll ships a stacked percentage bar (`.pollBar` / `.pollBarYes` /
`.pollBarNo`). The game reuses that idiom in two places (with their own classes,
so the features stay independent):

**Muster page (`/raid/`) — role-readiness meters** (`.roleMeter` / `.roleBar`):
team aggregate role rating vs. boss thresholds, green when met / red when not,
with the "need more healers!" nudge (the social-pressure messaging from spec
§5.3 is a UI responsibility — surface unmet thresholds prominently).

```
⚔️ Team Power 1,370   🛡️ Defense 770   ✚ Healing 340   👥 5
tank   [█████████░] 320 / 500 ✗  need more tanks!
healer [█████░░░░░] 280 / 300 ✗  need more healers!
dps    [██████████] 900 / 800 ✓
```

**Live battle (`/live/`) — boss + party HP bars** (`.hpBar`/`.hpBarFill`,
`.pmBar`/`.pmBarFill`): the boss bar depletes and party bars rise/fall as the
**combat log is revealed** turn by turn (see §8 — the log is the source of truth,
HP is recomputed from it). No "ticking down through the week" — HP only moves
during the raid-night battle.

Reuse the okra-campy palette / Comic Neue so it feels native; respect
`prefers-reduced-motion` (already wired) for the bar transitions and the reveal.

---

## 5. Accessibility, polish, correctness

- Bars need text equivalents (current/max numbers), not color alone, for the
  pass/fail thresholds — don't rely on red/green.
- `prefers-reduced-motion`: don't animate the HP tick for users who opt out.
- Escape any user-supplied strings (display names, class names) before injecting
  into the DOM — they originate from chat. Prefer `textContent` over `innerHTML`
  for those fields.
- Lazy-load the Firebase SDK / game JS on the game page only, so the rest of the
  site stays light.

---

## 6. Scaling the read path (know before launch, not POC)

Every viewer holding an `onValue` listener is a **live RTDB connection**. The
free (Spark) tier caps at **100 concurrent connections** — a popular stream can
brush that. Options, in order of effort:

1. **POC:** direct `onValue` reads — fine at small scale.
2. **Launch (cheap):** the bot periodically snapshots the read-state (boss HP,
   totals, leaderboard) to a single small **static JSON** the site `fetch`es and
   polls every N seconds — turns N listeners into N cheap CDN reads and sidesteps
   the connection cap entirely.
3. **Launch (managed):** move Firebase to the Blaze (pay-as-you-go) tier.

Pick (2) or (3) before any high-traffic launch; don't silently hit the cap.

---

## 8. The live battle as a "replay player" (don't reinvent the wheel)

The battle is a **deterministic, seeded, append-only event log** produced by the
backend (spec §5.8 / `IMPLEMENTATION.md §L`). The live page is just a **player**
over that log — the same pattern as game replays / battle protocols (e.g.
Pokémon Showdown). This is what makes it robust:

- **Timed reveal synced to `combat.startsAt`.** Each frame computes
  `visible = floor((now - startsAt) / MS_PER_EVENT) + 1` and renders
  `events[0..visible]`. So everyone sees the same moment, **latecomers catch up**,
  a refresh resumes correctly, and a finished battle is **re-watchable** — no
  per-client streaming state needed.
- **HP recomputed from the revealed slice.** The player replays `amount`/`kind`
  over starting HP to draw boss + party bars (prefers `*HpAfter` fields if the
  engine stamps them).
- **`prefers-reduced-motion`** → reveal everything at once (no paced animation).

**Combat-log event contract the UI consumes** (`raids/<s>/<w>/combat`):
```jsonc
{ seed, status:"live"|"done", startsAt, bossMaxHp,
  result: { downed, bossHpRemaining, mvp },
  log: {                          // KEYS MUST SORT NUMERICALLY (0,1,2,…) — not push-ids
    "0": { type:"start", text },
    "1": { type:"turn", n:1 },
    "2": { type:"action", side:"party"|"enemy", actor, actorName, ability,
           kind:"damage"|"aoe"|"heal"|"buff", target, targetName, amount, crit, text },
    "N": { type:"end", outcome:"victory"|"defeat", text } } }
```
The engine owns the rendered `text` (battle "voice"); the UI also uses the
structured fields for HP bars and per-side coloring.

## 7. Out of scope for this repo

- No secrets, service-account JSON, or bot tokens — those live only in the
  backend's runtime environment.
- No writes to authoritative game state (§1).
- No operational/deployment detail (where/how the bot runs) — that's a private
  backend concern.
