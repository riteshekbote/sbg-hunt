# SBG hunt KB (Sky Betting & Gaming, H1)
- 2026-08-15 FINDING (LOW) @ http://l.sbgmail.skybettingandgaming.com/ats/metrics — UNAUTHENTICATED Prometheus metrics on ASP.NET Core service (prometheus-net; 1687B; 21 metrics). Leaks: usf_data_read/write/delete_count_total (internal "USF" storage layer on AWS DynamoDB), dotnet_total_memory_bytes (201,646,760), process start time (epoch 1786500566 ≈ 2026-08-11), GC counts, threads (37), handles (1106), working set (711MB), virtual memory (2.48TB). Sibling: /ats/healthz → 200 "Healthy". App is Cheetah Digital email-infra service (404 page "Page not Found | Cheetah Digital"); sbgmail.skybettingandgaming.com → 301 cheetahdigital.com (retired product). Host discovered via sbg-recon pipeline artifact (SBG-live.txt). No secrets/user data — LOW/Info.
- 2026-08-15 CLOSED @ feeds estate: live/stage/perf/test-football-api.awsdev.feeds.skybet.com + staging/test-abelson-feed-api + players-runningball-test + apa.paddypower.com ALL resolve to single origin 49.44.79.236 (AWS ap-south-1, 2405:200:1607:2820:41::36) — VPC security-group locked (SYN dropped from our network). Not reachable.
- 2026-08-15 CLOSED @ build.sbftp + oauth.{test,loadtest,autotest}.sweeps.sbgservices.com — AWS eu-west-1 ELBs, nginx 520B "403 Forbidden" deny-all (IP allowlist). Not reachable.
- 2026-08-15 CLOSED @ api.sweeps.sbgservices.com — AWS API Gateway, every path 403 {"message":"Missing Authentication Token"} (IAM-locked/route-less). admin-api.{env}.sweeps → CloudFront+S3 404 "Not Found" (no public content). itv7/admin storybook S3 hosts → 403 CF origin error.
- 2026-08-15 CLOSED @ autodiscover.{sportinglife,skybettingandgaming}.com → 302 to autodiscover-s.outlook.com = Exchange Online (X-FEServer PN3PR01CA0153 O365 proxy) — no on-prem NTLM. m1/t1.mails.skybettingandgaming.com = retired Apache (404 all paths). regstat.* fleet = Apache 400 (unreachable/timeout from here).
- 2026-08-15 CLOSED @ sportinglife.com autodiscover — O365 (same pattern).
- 2026-08-15 STATE: 1,414 live hosts corpus (5 groups) harvested by cron pipeline (8 runs since 08-08); ~200 hosts probed this session; dominant pattern = Cloudflare bot-management 403 (Unavailable Page / Attention Required) on everything customer-facing; direct origins (ELB/S3/API-GW/VPC) locked or dead.
- 2026-08-15 FINDING (LOW) @ POST https://feedback.prod.sbftp.sbgservices.com/api/feedback — UNAUTHENTICATED feedback write API, no rate limit, no field validation. Schema walk: {} → 400 {"reason":"Missing product"}; {"product":"x"} → 400 {"reason":"Missing data"}; {"product":"x","message":"y"} → 201 {"status":"ok"}. 9 submissions accepted in rapid succession (no throttling). CORS: access-control-allow-origin: * + methods POST/OPTIONS/HEAD + Content-Type → ANY website can spam the queue via browser (no preflight). Same behavior on feedback.test.sbftp.sbgservices.com. Behind CloudFront. In scope (sbgservices.com). Impact: feedback-queue spam/pollution + potential stored-XSS-if-rendered (unproven, admin panel not reachable). PoC footprint: 9 benign {"product":"test"} records submitted. Low; Medium only if stored XSS demonstrated.
- 2026-08-15 CLOSED @ itv7.staging.sweeps.sbgservices.com — 403 but serves 355KB SPA shell (client-side app only, no data); admin/storybook S3 hosts → 403 (no public list); msoid.skybingo.com dead (404 "Not Found."); teleport.dev.rafflee.co.uk empty-404 app (no routes); rafflee S3 buckets private (AccessDenied).

## 2026-08-19 fresh recon + FINDING (api.s6.sbgservices.com Super6 API)
- Fresh corpus: subfinder 6538 unique subs (5 groups) / dnsx 1950 resolved / httpx 1340 live.
  Corpus files: /tmp/opencode/sbg-fresh/{subs_*,resolved.txt,live.txt}
- NEW FINDING (Medium) @ api.s6.sbgservices.com (in scope, CloudFront, Express):
  Broken object-level authorization on Super6 v2 API — see
  reports/2026-08-19_super6_api_idor.md. Key facts:
  * Public OpenAPI spec /v2/swagger.json (56KB) declares securitySchemes:[] on all 37 paths
  * /v2/round/{roundId}/user/{userId} (int userId, sequential) -> ANY user's predictions
    incl. OPEN round 1 pre-close (game-integrity); 404 for invalid = account existence oracle
  * /v2/score/leaderboard/user/{userId}?period=season -> full name (PII) + results, no auth
  * /v2/score/season/user/{userId} -> per-round season summary
  * Self-routes (/user/self*, /round/stats/user/self, /round/active/user/self) = 401
    without session (X-Cust-Id alone insufficient) = session middleware exists; the
    /user/{userId} family skips it
  * CORS: GET with Origin evil -> ACAO:* + allow-credentials:true + allow-methods
    GET,HEAD,OPTIONS,POST,DELETE,PUT + allow-headers incl X-Session-Id (browser-exploitable)
  * Demos: 22537542 (Jamie Carragher) + 22239878 (Super 6 Admin) both 200 with
    predictions; 10000001/11111111/99999999 -> 404
  * Active round: id 1, open, season 2026-27, 522,396 predictions total
  * S3 s6-prod-s3-content = not listable (no claim)
- Undocumented-but-live v2 routes found: /content/feature (feature flags), /content/video,
  /auth/codes (404 now). /score/leaderboard?period=round&id={n} -> 404 (closed-round
  leaderboard data rotated; old rounds "Round not found").
- Super6 web app bundle: /static/js/main.d8c94e68.js (604KB) — app uses Authorization +
  X-Session-Id headers; never calls /user/{userId} (only /user/self) — mobile-only surface.
- NEXT: HUMAN registers free Super6 account -> own userId via authenticated /user/self/info
  -> end-to-end unauth cross-read demo (best PoC for triager). Submit to H1 (Sky Betting &
  Gaming program).
- 2026-08-19 GITHUB-SOURCE IDOR COMPLETION: third-party repos ChrisMusson/Super6
  (IDs.csv = REAL player IDs 19166310 Aldo Seffi / 15977871 Chris Musson) +
  matbroughty/fourfold-react (independent no-auth confirmation). Regular-player
  full names + results readable unauth via /score/leaderboard/user/{id} — IDOR
  now fully proven end-to-end (report updated). Auth flow intel: skybet.com
  /secure/identity/m/login/super6 {username,pin} -> ssoToken -> "authorization:
  sso <token>" (in scope; not tested).
- 2026-08-19 cross-verification pass (triager hat): all Super6 claims re-confirmed
  live; h2h-opponent unauth = REFUTED (returns default pundit Carragher = public by
  design — killed); older findings re-verified STILL LIVE (l.sbgmail Prometheus
  200; feedback 400 Missing product). Verdict: submit as Medium, expect
  Low/N-A debate on leaderboard-names-by-design; lead with open-round predictions
  + staff account + oracle + CORS credentials.
