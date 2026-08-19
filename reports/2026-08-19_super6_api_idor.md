# SBG FINDING DRAFT — Super6 API broken access control (api.s6.sbgservices.com)

## Status: VALIDATED-READY (H1 report draft — awaiting HUMAN submission)
## Date: 2026-08-19 (fresh corpus via subfinder 6538 subs / 1340 live / 1950 resolved)

## Title
Super6 API exposes any user's predictions, results and names via unauthenticated
sequential integer userId routes (broken object-level authorization + account
existence oracle + permissive CORS)

## Host
- https://api.s6.sbgservices.com (in scope: sbgservices.com; CloudFront; Express backend)
- API version v2; OpenAPI 3.0 spec PUBLIC at https://api.s6.sbgservices.com/v2/swagger.json
  (56,314B; title "Super6 API"; **securitySchemes: []** — spec declares ZERO auth on all 37 paths)
- Docs landing: https://api.s6.sbgservices.com/v2/docs/

## Endpoints (all GET, no auth, no session, no API key)
1. GET /v2/round/{roundId}/user/{userId}
   - userId schema: integer, pattern ^[0-9]+$ (sequential, enumerable)
   - 200 -> {"roundId","hasPredicted","hasEnteredHeadToHead","predictions":{"scores":[{challengeId,scoreHome,scoreAway,matchId,...}]}}
   - Returns the user's EXACT score predictions for the round — including the OPEN
     round (roundId=1, status "open", closes 2026-08-22) = pre-close prediction copy
     (game integrity) + opponent-advantage.
   - 404 (0B) for non-existent IDs -> account existence oracle.
2. GET /v2/score/leaderboard/user/{userId}?period=season
   - 200 -> {"userId","position","firstName","lastName","isWinner","goldenGoal",
     "correctResults","correctScores","points","isMasked"} — full name (PII) + results.
3. GET /v2/score/season/user/{userId} — per-round season score summary (404 when no data).

## Proof (real IDs, 1rps)
- userId 22537542 ("Jamie Carragher", pundit): /round/1/user/22537542 -> 200 predictions
  (hasPredicted:true, 6 challenges); /score/leaderboard/user/22537542?period=season -> 200
  {firstName:"Jamie",lastName:"Carragher",isMasked:false,...}
- userId 22239878 ("Super 6 Admin" — internal staff account): /round/1/user/22239878 ->
  200 predictions (hasPredicted:true); leaderboard/user -> 200 {firstName:"Super 6",lastName:"Admin"}
- Negative: userIds 10000001/11111111/99999999 -> 404 (0B) — valid-account oracle
- Self-contrast: /v2/round/stats/user/self, /v2/user/self/info, /v2/round/active/user/self
  -> ALL 401 without session (even with X-Cust-Id: 22537542) — the API HAS session
  middleware; the /user/{userId} family simply skips it.

## CORS (browser-exploitability)
- GET /v2/round/1/user/22537542 with Origin: https://evil.example.com -> 200 with
  access-control-allow-origin: *, access-control-allow-credentials: true,
  access-control-allow-methods: GET,HEAD,OPTIONS,POST,DELETE,PUT,
  access-control-allow-headers: Authorization,X-Cust-Id,Content-Type,X-Session-Id
- OPTIONS preflight with evil Origin -> ACAO: https://super6.skysports.com (whitelist)
  — but plain GETs bypass preflight (simple request) and are readable cross-origin.
- Any attacker website can fetch arbitrary users' predictions/names from visitors' browsers.

## Supporting evidence
- Spec publicly served (swagger.json) with securitySchemes:[] — no auth documented.
- /v2/content/feature + /v2/content/video = additional undocumented-but-live routes (200).
- S3 bucket s6-prod-s3-content (assets in round payloads) = NOT listable (AccessDenied) — no claim.
- Active round: id=1, status "open", season 2026-27, endDateTime 2026-08-22T14:00:00Z;
  prediction-count endpoint: {"roundId":1,"total":522396} (522k predictions — scale of exposure)

## Impact
- Broken object-level authorization: any user's current-round predictions (incl. open
  round pre-close), results, season scores and full name readable without auth by
  numeric-ID enumeration.
- Game-integrity: open-round prediction copying / head-to-head opponent prediction
  preview (the endpoint description says opponent info is shown "if the info is
  available" — it is available pre-close via this route).
- PII: full firstName/lastName for any account (privacy masking is client-side:
  isMasked flag returned but data included).
- Account existence oracle over the 8-digit space (feeds targeted phishing/abuse).

## Severity: Medium (CVSS ~5.3 AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N) — argue toward
## Medium-High if open-round prediction integrity accepted as game manipulation.

## Notes for submission
- PoC footprint: ~15 GET requests at 1rps, two real IDs (pundit + staff) + 3 invalid;
  no user data stored locally.
- userId 22239878 = "Super 6 Admin" — internal account data exposure = strong demo.
- Regular-player ID unavailable passively (leaderboards empty until round 1 closes
  2026-08-22; winners[] empty). Option: HUMAN registers free Super6 account -> get own
  userId -> full end-to-end demo (own ID via authenticated /user/self/info, then
  cross-check unauth read).
- OpenAPI spec itself is public (swagger.json) — include URL as proof-of-design.
## UPDATE 2026-08-19 (GitHub-source IDs) — REGULAR-PLAYER IDOR PROVEN
- GitHub code search found third-party clients:
  * ChrisMusson/Super6 (github.com/ChrisMusson/Super6) — get_ids.py reveals the
    official auth flow: POST https://www.skybet.com/secure/identity/m/login/super6
    {username,pin} + X-Requested-With: XMLHttpRequest -> user_data.ssoToken ->
    "authorization: sso <token>" header (skybet.com/secure/identity = IN SCOPE).
    Repo IDs.csv = two REAL regular-player userIds: 19166310, 15977871.
  * matbroughty/fourfold-react — independent confirmation (2026-08-08): spec at
    /v2/swagger.json, swagger UI = unconfigured petstore boilerplate, "no credentials,
    no cookies, no API key" needed; round ids restart every season; only current
    season served.
- PROD PROOF on regular players (1rps):
  * GET /v2/score/leaderboard/user/19166310?period=season -> 200
    {"userId":19166310,"position":0,"firstName":"Aldo","lastName":"Seffi",...,"isMasked":false}
  * GET /v2/score/leaderboard/user/15977871?period=season -> 200
    {"userId":15977871,"position":0,"firstName":"Chris","lastName":"Musson",...}
  * GET /v2/round/1/user/{both} -> 404 (not entered round 1; 200 when entered —
    predictions exposure proven via pundit/staff IDs)
- Impact statement now: ANY registered Super6 player's full name + results + season
  scores readable by sequential integer userId, no auth; combined with 200/404
  existence oracle over the 8-digit space; CORS * makes it browser-exploitable.
- auth-flow intel (KB only, NOT tested): skybet.com/secure/identity/m/login/super6
  JSON login (username+pin, X-Requested-With) -> ssoToken; sso auth header scheme.

## TRIAGER-VERDICT PASS + FULL CROSS-VERIFICATION (2026-08-19)
All claims re-tested today, 1rps, in-scope host api.s6.sbgservices.com:
| Claim | Status |
|---|---|
| Spec securitySchemes empty / no per-path security | CONFIRMED (all 37 paths) |
| 200-vs-404 account oracle (4 real, 3 fake IDs) | CONFIRMED |
| /round/{id}/user/{userId} open-round predictions unauth | CONFIRMED (Carragher, Super6 Admin) |
| /score/leaderboard/user/{id}?period=season -> full name+results | CONFIRMED (4/4 real IDs: Carragher, Admin, Aldo Seffi, Chris Musson) |
| /score/season/user/{id} season summary | CONFIRMED |
| CORS ACAO:* + credentials:true on GET | CONFIRMED |
| Self-routes /user/self* etc = 401 (middleware exists) | CONFIRMED (4 endpoints) |
| h2h-opponent unauth | REFUTED as finding: returns default pundit (Carragher) = public content by design |
| Older findings l.sbgmail Prometheus + feedback API | STILL LIVE (re-verified today) |

TRIAGER VERDICT (honest):
- Acceptable as Medium at best. Strongest claims: (a) open-round predictions of
  arbitrary users pre-close incl. internal staff ("Super 6 Admin"), (b) existence
  oracle over 8-digit space, (c) isMasked flag exists (privacy control) but returns
  isMasked:false for everyone tested, (d) session middleware provably exists on
  self-routes — the user/{userId} family simply omits it.
- Weak spots a triager will hit: leaderboard names are displayed publicly in-app
  (top players visible) -> may argue name exposure = by design; regular-player
  open-round predictions NOT demoed (known regular IDs not entered round 1 yet);
  spec is public + no auth declared -> "you didn't need to guess".
- Realistic outcome: Accepted Low-Medium. If rejected: "public data / by design"
  is the likely N/A rationale; the staff-account (22239878) read + predictions
  pre-close + CORS credentials:true are the strongest counterpoints.
- Recommendation for submission framing: BOLA on user-scoped endpoints; lead with
  open-round predictions (integrity) + staff account + oracle; present leaderboard
  names as secondary; note masking flag; add CORS * as amplifier; declare read-only
  testing, no data retained beyond two IDs published by a public repo.

## FINAL STATUS: OUT OF SCOPE — NOT SUBMITTED (2026-08-19)
- HackerOne "Sky Betting & Gaming" program: *.s6.sbgservices.com = OUT OF SCOPE (confirmed by user on live program page).
- Bugcrowd "Flutter UK&I" VDP (bugcrowd.com/flutter-uki): scope = *.sbgservices.com IN, but
  *.s6.sbgservices.com EXPLICITLY listed in the Out-of-Scope section (also OOS: *.sbgmail.*,
  super6.skysports.com, *.tradingmodels.io).
- No alternative channel (security.txt absent; program terms require platform-only disclosure).
- VERDICT: finding dropped (valid vuln, untestable-scope). Evidence kept in this file for
  future scope changes. NOTE: l.sbgmail Prometheus metrics finding from 08-15 = also OOS
  (*.sbgmail.skybettingandgaming.com OOS on Bugcrowd) — check if that report was filed on H1.
- INTEL KEEPER (in scope, untested): skybet.com/secure/identity/m/login/super6 ssoToken
  flow (skybet.com IN scope on Bugcrowd *.skybet.com) — future lead for SBG on Bugcrowd.
