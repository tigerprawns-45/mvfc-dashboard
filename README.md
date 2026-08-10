# Manly Vale FC — Results & Ladders

A single-page dashboard showing every competitive Manly Vale FC team (U12 and up,
mixed and women's): current ladder position, played/won/drawn/lost with goals for,
against and difference, recent form with scores, latest result, and the next fixture
with time and ground. Data comes live from the MWFA Dribl match centre.

Once any team is starred the four grade groups open collapsed, so the board is a
short index to choose from rather than 38 cards to scroll past, with `Your teams`
expanded above them. With nothing starred they open expanded — there would be
nothing else on the page otherwise.

`index.html` is the whole application. No build step, no dependencies, no server-side
component.

## Running it

It must be served over `http://` or `https://` — **not** opened from disk.

```bash
python3 -m http.server 8000
```

Then visit <http://localhost:8000/>.

Opening `index.html` directly as a `file://` URL works in Chrome but fails in every
version of Safari, which blocks cross-origin requests from a null origin regardless
of how the remote server is configured. This is the single most likely cause of a
"Failed to fetch" error.

## Hosting

Any static host works. For GitHub Pages:

1. Push this repository to GitHub.
2. **Settings → Pages → Build and deployment → Source: Deploy from a branch**,
   branch `main`, folder `/ (root)`.
3. The dashboard appears at `https://<user>.github.io/<repo>/`.

No Actions workflow and no scheduled job are needed — the page fetches Dribl directly
in the browser, so it is always current and the Refresh button re-pulls on demand.

## How it works

Dribl exposes no way to list a club's teams. Instead the page:

1. Lists the season's competitions, then each competition's leagues.
2. Keeps leagues with `ladder_access === "public"`. This is exactly the set of
   competitive grades — U06–U11 collect results but don't publish ladders, so they
   are `"body"` and drop out. No age list is hardcoded, so regrades and new divisions
   are picked up automatically.
3. Fetches each public division's ladder and keeps rows where `club_code === "MVFC"`.
   Each ladder row carries `recent_matches` and `upcoming_matches` alongside the
   table stats, so one call yields position, form, last result and next fixture.
4. Fetches the club fixture list for grounds, which the ladders feed does not carry,
   and joins it on division name + exact kick-off time.

A full scan is about 90 requests, run six at a time.

### Division shortlist

Only 36 of this season's 75 public divisions contain a Manly Vale team; the other 39
are fetched and discarded. So the divisions that did contain one are remembered in
`localStorage` under `mvfc:divs:v<cacheVersion>:<season-id>`, and an ordinary visit
scans just those — 48 requests instead of 90, since the competition and league
listings are skipped along with them.

The risk is a team appearing in a division the shortlist has stopped looking at: a
regrade, a late entry, a finals draw moving a team up. Three things catch it.

- **Refresh always rescans everything.** It is the manual escape hatch.
- **The list expires after `CFG.divisionTtlDays`** (7), so a stale shortlist can
  never be more than a week old.
- **A shortlist scan that finds nothing immediately runs a full one** rather than
  reporting no teams — this is what absorbs a season rollover or a wholesale regrade.

Only a full scan writes the list, so a run of narrow loads can't keep postponing the
weekly rescan.

### Saved copy

The last successful result is kept in `localStorage` and painted immediately on the
next visit — about 240 ms to first card, against several seconds for a cold load —
then replaced when the live data arrives. If Dribl can't be reached, the saved copy
stays on screen with a warning rather than the page failing empty.

Only rendered fields are stored, which keeps it near 175 KB. Storing the raw ladder
rows would blow the quota: each row carries `recent_matches` and `upcoming_matches`
for every team in the division, none of which is displayed. Bump `CFG.cacheVersion`
after changing the stored shape.

A bump changes the key rather than the value, so the copy it supersedes would sit
there indefinitely. `pruneOldCaches()` clears any `mvfc:v*` / `mvfc:divs:v*` key that
isn't the current one on startup, which also collects last season's. Favourites are
excluded by design — that key is unversioned and holds the only thing in storage a
person chose by hand.

Writes are wrapped in `try`/`catch` because Safari throws on `localStorage.setItem`
in Private Browsing. Caching is an optimisation; nothing depends on it succeeding.

### Favourites

Starring a team pins it to a `Your teams` section above the board and to the top of
Next up. The selection lives in `localStorage` and in a `?fav=` URL parameter, so a
link can be shared and opens focused on the same teams. See
[FAVOURITES.md](FAVOURITES.md) for the full behaviour and the reasoning behind the
less obvious choices.

### Season rollover

Update `CFG.season` in `index.html` when the season changes. The current value is
Winter 2026. Everything else — tenant, club, division list — is either fixed or
discovered at runtime.

## Notes on the Dribl API

Base `https://mc-api.dribl.com/api`, public, no auth.

- `access-control-allow-origin: *` on every endpoint, which is what makes the
  browser-side approach viable.
- Cloudflare rejects requests with a non-browser `User-Agent` with a 403. Browsers are
  unaffected; command-line testing needs a real UA string.
- `/fixtures` ignores `disable_paging` and is cursor-paginated 30 at a time. The
  cursor is plain base64 JSON, so the page hand-builds one starting eight days back
  rather than crawling from round one — about 12 pages instead of 60. It falls back to
  normal paging if the seeded cursor is ever rejected.
- Hash IDs are **not** shared between `/fixtures` and `/ladders`; the same match has
  different identifiers in each. Hence the join on division + timestamp.
- The two feeds use different timestamp formats, and `/fixtures` sends six
  fractional-second digits, which Safari's date parser rejects outright. Both formats
  funnel through the single `when()` helper — do not parse a Dribl date any other way.
- Unallocated fixtures ship with an empty team name and a null ground. These render as
  "opponent TBC" and "Ground TBC"; they are not lookup failures.

## Files

| File | Purpose |
|---|---|
| `index.html` | The dashboard. Everything lives here. |
| `FAVOURITES.md` | Behaviour and rationale for the favourites feature. |
| `.claude/launch.json` | Local preview server config. |
