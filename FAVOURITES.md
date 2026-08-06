# Favourites — specification

Status: **proposed, not built.** Written for review before implementation.

Lets someone mark the teams they care about so those teams sit at the top of the
board and the top of Next up, and so the choice survives a reload and can be shared
by link.

---

## 1. Behaviour

### Marking

Every team card carries a star control. Tapping it toggles that team's favourite
state. The change takes effect immediately — no save step, no confirmation.

### Ordering

Favourited teams are **pinned, never filtered.** A shared link opens on someone
else's phone showing their friend's teams first and everything else still below,
so the page never looks broken or truncated to a person who didn't create the link.

**Board.** A new section, `Your teams`, renders above all four existing group
headings. It contains every favourited team, in the same sort order they would
otherwise have had. Favourited teams are *moved*, not duplicated — they no longer
appear in `Junior mixed` and friends. Group counts adjust accordingly, and a group
whose teams are now all favourited is omitted entirely rather than rendered empty.

`Your teams` is collapsible like the others but always defaults to open, including
on mobile.

**Next up.** Favourited fixtures keep their chronological position within the list
but the panel gains a `Your teams` block at the top, repeating just those fixtures
in time order. Here duplication *is* correct: the whole point of Next up is the
chronological sweep, and removing entries from it would break that reading.

Cap the pinned block at the next 5 favourite fixtures so it can't dominate the panel.

### Empty state

With nothing favourited, the `Your teams` section and the pinned Next up block are
both absent — no placeholder, no empty heading. The page looks exactly as it does
today. The stars are the only new pixels.

---

## 2. Storage

`localStorage`, key `mvfc:favs:<season-id>`, value a JSON array of `team_hash_id`
strings.

**Scoping the key to the season is deliberate.** `team_hash_id` is the only stable
team identifier Dribl exposes and it is allocated per season — the same real-world
team gets a different hash next winter. Including the season id means a rollover
starts clean instead of silently matching nothing, or worse, matching a stranger's
team that inherited the hash. It also matches how people actually use this: a child
changes division every year, so last season's favourites are the wrong answer anyway.

Old seasons' keys are left in place. They are a few hundred bytes and deleting them
risks throwing away a season a user may still switch back to.

**`localStorage` must be treated as failable.** Safari throws on write in Private
Browsing rather than returning an error. Every read and write goes through
`try`/`catch`; on failure favourites degrade to in-memory only for that session and
nothing else on the page breaks.

---

## 3. Shareable links

`?fav=<hash>,<hash>,<hash>`

**Reading.** If the parameter is present it takes precedence over `localStorage` for
that page load, and is then written to `localStorage` so a reload without the
parameter keeps the same selection. Hashes in the parameter that match no team this
season are ignored silently — a stale link degrades to fewer teams, not an error.

**Writing.** Toggling a star updates the URL in place via `history.replaceState`, so
the address bar is always shareable and the back button is unaffected. When the last
favourite is removed the parameter is dropped rather than left empty.

A `Copy link` control sits in the `Your teams` heading, visible only when at least
one team is favourited. It uses `navigator.clipboard` with a select-the-text
fallback, since clipboard access needs a user gesture and a secure context — both
satisfied here, but iOS has historically been inconsistent.

---

## 4. Interaction and markup

### The nested-button problem

Today the entire card header is one `<button class="team-head">`. A `<button>`
cannot legally contain another `<button>`; browsers recover inconsistently and iOS
Safari in particular mis-targets the tap.

So the card header is restructured:

```
.team
  .team-top                 <- flex row, not interactive itself
    button.team-head        <- expands ladder; everything except the star
    button.star             <- toggles favourite
  .ladder                   <- unchanged
```

`.team-head` keeps its current content and `aria-expanded`. `.star` is a sibling.
This is the only structural change to existing markup, and it is required — it is
not incidental refactoring.

### The star control

- Minimum 44×44 px hit area, per iOS guidance, achieved with padding rather than
  icon size so the visual weight stays small.
- Positioned top-right of the card, vertically aligned with the ladder position
  number, which moves left to make room.
- Filled `★` in `--amber` when on; outlined `☆` in `--ink-faint` when off.
- `aria-pressed` reflects state. `aria-label` reads
  `Favourite <division>` / `Unfavourite <division>`.
- No transition on the fill, or it feels laggy on a slow phone. A short scale
  bounce is fine and is suppressed under `prefers-reduced-motion`, which the
  stylesheet already honours globally.

### Re-render on toggle

Toggling reorders the board. Re-rendering the whole board on every tap is simplest
and, at 38 cards, fast enough — but it collapses any expanded ladder and loses
scroll position, which is jarring.

**Preferred:** move the card's DOM node between sections and update the Next up
pinned block, leaving everything else untouched. Slightly more code, no visual
disruption. If that proves fiddly, full re-render with scroll position restored is
an acceptable fallback.

---

## 5. Edge cases

| Case | Behaviour |
|---|---|
| Two Manly Vale teams in one division (`AL 04 Mixed`, `W-AL 03 Female` this season) | Starred independently; the hash distinguishes them. The card heading must show enough to tell them apart — currently both cards read the same division name, so the team's own suffix needs surfacing. |
| Favourite has no upcoming fixture | Appears in `Your teams` on the board, absent from the pinned Next up block. |
| Every team favourited | `Your teams` holds all 38, the four group headings vanish. Allowed. |
| `?fav=` with 100 junk values | Ignored; only hashes matching a loaded team count. |
| Favourites set, then Dribl fails to load | Cached data still renders with favourites applied — the two features compose. |

---

## 6. Out of scope

- Any server-side or cross-device sync. The link is the sharing mechanism.
- Reordering favourites by hand. They keep the standard sort.
- Notifications for favourited teams. Needs a backend; see README rationale.

---

## 7. Open questions

1. Should `Your teams` also pin to the top of the **tally** — i.e. a W/D/L summary
   for favourites only, alongside the club-wide one? Useful for a parent with two
   kids playing, noise for everyone else. Leaning no.
2. Should the star appear in the Next up rows too, or only on cards? Leaning cards
   only — two places to toggle the same state invites confusion about scope.
