# Manly Vale FC — Results & Ladders

A single-page dashboard showing every competitive Manly Vale FC team (U12 and up,
mixed and women's): current ladder position, recent form with scores, latest result,
and the next fixture with time and ground. Data comes live from the MWFA Dribl match
centre.

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

That is roughly 90 requests per load, run six at a time.

### Saved copy

The last successful result is kept in `localStorage` and painted immediately on the
next visit — about 240 ms to first card, against several seconds for a cold load —
then replaced when the live data arrives. If Dribl can't be reached, the saved copy
stays on screen with a warning rather than the page failing empty.

Only rendered fields are stored, which keeps it near 160 KB. Storing the raw ladder
rows would blow the quota: each row carries `recent_matches` and `upcoming_matches`
for every team in the division, none of which is displayed. Bump `CFG.cacheVersion`
after changing the stored shape.

Writes are wrapped in `try`/`catch` because Safari throws on `localStorage.setItem`
in Private Browsing. Caching is an optimisation; nothing depends on it succeeding.

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
| `FAVOURITES.md` | Spec for the favourites feature. Proposed, not yet built. |
| `.claude/launch.json` | Local preview server config. |
