---
name: MVFC Dashboard
id: mvfc-dashboard
stage: live
owner: Jason
summary: Live results, ladders, recent form and next fixtures for every competitive Manly Vale FC team (U12 and up, mixed and women's), read straight from the MWFA Dribl match centre in the browser — one self-contained index.html on GitHub Pages, no build step, no backend, no scheduled job.
links:
  live: https://tigerprawns-45.github.io/mvfc-dashboard/
  repo: https://github.com/tigerprawns-45/mvfc-dashboard
---

Built for parents at a ground on a Saturday morning, which is the constraint that decides
most things here: mobile is the primary device, reception is unreliable, and the question is
almost always "where and when is my kid playing, and how did they go last week". Each team
gets a card with ladder position, the season record with goals for, against and difference,
the last five results with scores, the latest result and the next fixture with time and
ground; tapping it opens the full division ladder and the last five in detail. A
chronological Next up panel runs across every team's next match. Starring a team pins it
above everything and into a `?fav=` link that can be texted to another parent — and it also
collapses the four grade groups, so the board turns from 38 cards into your teams above an
index. Star nothing and it stays a board, because four closed headings would be no page at
all.

The whole application is `index.html` — no build step, no dependencies, no server-side
component. That is a decision rather than an accident. The Dribl match centre API is public
and sends `access-control-allow-origin: *`, so the browser can fetch it directly from a
static origin, which means nothing to schedule, nothing to deploy on a cycle, and no
possibility of the data being stale because a cron failed quietly. The cost is 90 requests on
a first load, mitigated by painting from a saved copy first and by remembering which 36 of
the 75 public divisions we actually play in — an ordinary visit costs 48.

Teams are discovered rather than configured: Dribl exposes no way to list a club's teams, so
the page scans every public league's ladder for the club code. Filtering on
`ladder_access == "public"` happens to be exactly the definition of "competitive" — U06–U11
collect results but publish no ladder — so new grades and regrades are picked up with no
hardcoded age list to maintain.

[README.md](README.md) documents the API's traps and the caching design.
[FAVOURITES.md](FAVOURITES.md) carries the favourites behaviour and the reasoning behind the
non-obvious calls.

## Roadmap

### Decisions

- [ ] **Is this a personal tool or the club's dashboard?** It currently lives on a personal
  GitHub account at `tigerprawns-45.github.io`, and the club badge is hot-linked from Dribl's
  CDN. If the club adopts it the URL moves, and that is not free: favourites and the cached
  copy both live in `localStorage`, which is scoped to the origin, so every user silently
  loses their starred teams and their saved data on the move. Doing it before finals is
  cheaper than after. Blocks any decision about branding, a custom domain, or telling parents
  to bookmark it.

### Build

- **Detect the current season instead of hard-coding it.** `CFG.season` is pinned to Winter
  2026. `/api/list/seasons` returns `is_current`, so this can resolve itself. Left alone, next
  winter the dashboard does not fail — it quietly serves 2026 ladders to people reading them
  as current, which is worse than an error page.
- **Add to Home Screen.** A web manifest and an `apple-touch-icon` give a real icon and a
  standalone launch on iOS. Cheap, and it makes the thing easier to come back to — which
  matters more than it looks, given the storage eviction note below.
- **A "Today" strip on match days.** Next up is chronological, which buries the only thing
  anyone wants at 8am on a Saturday: my teams, today, what time, which field. Worth building
  only after favourites have been used in anger for a few weekends — the strip is meaningless
  without them.

### Noted, not scheduled

- **There are no Manly Vale women's over-35 or over-45 teams to find.** Checked against
  every division in both competitions, public ladder or not: 38 Manly Vale rows, all under
  club code `MVFC`, none in a women's over-age grade. MWFA publishes no `W-O35` or `W-O45`
  at all — the women's over-age competition is `W-40`, and the club enters no team in it.
  The five over-age sides it does field are all mixed, and all five are on the board. Worth
  recording because the absence looks exactly like a scan that is silently dropping teams,
  and the discovery is deliberately unconfigured — there is no list to check a suspicion
  against. Re-run the club-code sweep before assuming a bug next time.
- **The Safari failures were `file://`, not CORS.** Dribl sends `access-control-allow-origin:
  *` on every endpoint; Safari blocks cross-origin fetch from a null origin regardless of what
  the server says, while Chrome permits it. Hosting on a real origin fixed it, confirmed by
  testers. Recorded because the symptom — "works on Android Chrome, fails on every iPhone" —
  invites a rewrite that was never needed.
- **Architecture B is not needed and would be worse.** A Node prefetch on a cron writing a
  static `data.json` was the fallback if CORS had been closed. It isn't. B costs freshness and
  adds a scheduled job to maintain, and it would have run straight into the Cloudflare
  user-agent trap documented in [README.md](README.md). Do not revisit without a new reason.
- **No push notifications.** iOS web push requires the user to install the PWA first, and
  reliable delivery needs a server and a scheduler — which reintroduces exactly the
  infrastructure the current architecture exists to avoid, for something nobody has asked for.
- **iOS Safari deletes `localStorage` after 7 days without a visit.** Favourites and the
  cached copy both go, so a fortnightly user returns to no starred teams and a cold load.
  Nothing to fix in the app — it is the reason the `?fav=` link matters, since a bookmark
  survives eviction where the storage does not.
- **This project does not express the portfolio thesis, and shouldn't be made to.** It makes
  no predictions and no judgments; it reports facts from an external source, so there is
  nothing here for an AI to grade itself on. The one honest connection is the second rule in
  miniature — "Ground TBC" and "opponent TBC" where Dribl has no data, a past match's venue
  left blank rather than guessed, favourite hashes never pruned because one transient fetch
  error would delete real data. Naming the gap rather than papering over it. That is real but
  small, and inflating it into a thesis story would be the exact failure the thesis warns
  about.
