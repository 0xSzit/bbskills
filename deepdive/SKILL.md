---
name: deepdive
description: Use when deep-diving a live, functioning web application (Tier 1 target) for manual testing — asked to "map this app", "test this subdomain", "deep dive on X", "analyze JS files", or "find hidden endpoints". Prompts the user to pick one subdomain from the active/live list before proceeding. Covers Burp-logged browsing, test accounts, JS analysis, and endpoint discovery.
---

# Application Mapping & Deep Testing (per Tier-1 target)

Run this once per prioritized application from `tier1_apps.txt`. Confirm this specific
host is in scope (not just the root domain) before testing.

## 0. Select the target app
Before doing anything else, read `tier1_apps.txt` (or `live/httpx.json` filtered to
Tier 1) and present the live, running apps as a numbered list — don't just pick one.
Include status code, title, and tech stack per line so the choice is informed, e.g.:
```
1) api.target.com      [200]  "Target API Gateway"   tech: nginx, Kong
2) app.target.com      [200]  "Target Dashboard"      tech: React, AWS CloudFront
3) portal.target.com   [302]  "Login - Partner Portal" tech: PHP, Apache
```
Ask the user which one to deep-dive (or accept them naming it directly, e.g. "deep
dive app.target.com"). If no `tier1_apps.txt` exists yet, say so and point back to
the `scopehawk` skill instead of guessing a target.

Once chosen, set `TARGET_APP` to that host and use it for every command below —
don't silently default to the first entry in the list. If the user wants more than
one app covered, run this whole skill once per app, sequentially, confirming between
each ("done with app.target.com — move to portal.target.com next?").

## 1. Set up logging
- Burp Suite running, proxy on, target scoped to this host only.
- Turn on Burp's "Logger" / save full history for this app.
- Browser through Burp proxy (or FoxyProxy profile per-target).

## 2. Create two test accounts
Register two distinct users (e.g. `tester1`, `tester2`) if signup is open — this is
required for IDOR/BAC/privilege-escalation testing later (compare what user A can
see/do vs user B). Note: differing roles (user vs admin) if the app supports tiers.

## 2b. Capture auth state once, reuse everywhere
Grab each account's session cookie or bearer token right after login and keep both
on hand for the rest of this skill — don't re-login by hand for every tool:
```bash
export AUTH_A="session=<tester1_cookie>"
export AUTH_B="session=<tester2_cookie>"
```
Feed `-H "Cookie: $AUTH_A"` (or `Authorization: Bearer ...`) into every httpx/katana/
ffuf/nuclei/dalfox call below — anonymous recon misses most IDOR, BAC, mass-
assignment, and privilege-escalation bugs because they only exist behind login.
Never print full cookies/tokens into shared notes or commit them anywhere — redact
to a short hash if you need to reference which session a finding came from.

**Automated A/B diffing** (catches IDOR/BAC you'd otherwise only find by luck):
hit the same set of endpoints as both accounts and diff the responses —
status code, body length, and specific fields that shouldn't match:
```bash
while read url; do
  a=$(curl -s -o /dev/null -w "%{http_code} %{size_download}" -H "Cookie: $AUTH_A" "$url")
  b=$(curl -s -o /dev/null -w "%{http_code} %{size_download}" -H "Cookie: $AUTH_B" "$url")
  [ "$a" != "$b" ] && echo "[DIFF] $url  A=$a  B=$b"
done < map/sitemap.txt
```
Any diff on an endpoint that *should* behave identically for both accounts (e.g.
both hitting `/api/profile` but getting different-sized bodies back for what
should be symmetric data) is worth a manual look.

## 3a. Note-taking (keep this running the whole session — don't skip it)
```markdown
# TARGET: app.target.com -- SESSION 1

## Interesting Leads (not confirmed yet)
- [14:22] /api/v2/invoices/{id} -- no auth check visible, testing...

## Dead Ends (don't revisit)
- /admin -> IP restricted, confirmed with 15+ bypass headers tried

## Anomalies
- GET /api/export returns 200 even with session cookie removed
- POST /api/check-user -> 150ms (exists) vs 8ms (doesn't) -- enumeration oracle

## Confirmed Bugs
- [15:10] IDOR on /api/invoices/{id} -- read+write, both accounts tested
```
This matters more than it looks — anomalies you don't write down the moment you
see them get lost in a long session, and "dead ends" logged properly stop you
re-testing the same blocked path three separate times.

## 3. Map the application
Browse every feature by hand — nav menus, settings, account pages, admin areas,
file upload, search, checkout/payment flows, API-backed SPA routes. Goal: exhaustive
sitemap + understanding of business logic (what *should* and shouldn't be possible).

Supplement manual mapping with a crawler, but don't rely on it alone:
```bash
katana -u https://$TARGET_APP -jc -kf all -d 5 -c 10 -o map/katana.txt
hakrawler -url https://$TARGET_APP -depth 3 -plain >> map/hakrawler.txt
```

## 4. JS file analysis
Pull every JS file the app loads, then mine it:
```bash
cat map/katana.txt | grep "\.js$" | sort -u > map/js_files.txt
while read url; do curl -s "$url" >> map/all_js_combined.js; done < map/js_files.txt

# Endpoints/paths hidden in JS
cat map/all_js_combined.js | grep -oE "[\"'](\/[a-zA-Z0-9_\-\/\.]+)[\"']" \
  | tr -d "\"'" | sort -u > map/js_endpoints.txt

# Or use purpose-built tools:
python3 LinkFinder/linkfinder.py -i map/all_js_combined.js -o cli >> map/js_endpoints.txt
nuclei -l map/js_files.txt -tags exposure,token -o map/js_secrets.txt   # API keys/tokens in JS
```
Look specifically for: API base URLs, internal/staging hostnames, admin routes,
feature flags, hardcoded credentials/keys, GraphQL endpoints, source maps (`.js.map`
→ can reconstruct original source).

## 5. Historical & external endpoint discovery
```bash
echo "$TARGET_APP" | gau --subs | anew map/historical_urls.txt
echo "$TARGET_APP" | waybackurls | anew map/historical_urls.txt
# Filter for interesting params/endpoints
cat map/historical_urls.txt | gf xss >> map/candidates_xss.txt
cat map/historical_urls.txt | gf sqli >> map/candidates_sqli.txt
cat map/historical_urls.txt | gf ssrf >> map/candidates_ssrf.txt
```

## 6. API spec / hidden route discovery
```bash
# Common paths worth always checking
ffuf -u https://$TARGET_APP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt -mc 200,201,301,302,401,403
# swagger/openapi/graphql introspection
curl -s https://$TARGET_APP/swagger.json
curl -s https://$TARGET_APP/.well-known/openapi.json
curl -s -X POST https://$TARGET_APP/graphql -d '{"query":"query{__schema{types{name}}}"}' -H "Content-Type: application/json"
```

## 7. Now actually test — per-category procedures

Work through every category below for this app. For each, the goal is a concrete
pass/fail signal, not a vague "looked okay." Log every test (request, response,
verdict) even when it fails — negative results matter for coverage.

### 7a. Authentication
- **Username/email enumeration**: compare responses for valid vs invalid
  username on login, register, and forgot-password endpoints — diff status code,
  body length, response time. Any difference = enumeration.
- **Password policy / brute force**: check if there's a lockout after N attempts;
  if not, that's a finding on its own. Test credential stuffing resistance with
  Burp Intruder (sniper on password field, pitchfork for user:pass pairs) at low
  rate — stop the moment you've proven the point, don't actually brute a real pass.
- **Password reset**: intercept the reset-token request. Is the token short, a
  timestamp, a sequential ID, or a predictable JWT? Does the token expire? Can the
  same token be reused twice? Does the reset endpoint accept a token for a *different*
  user's account if you swap the `email`/`user_id` param while keeping your own
  valid token (token-user binding bypass)?
- **OTP/2FA**: is there a rate limit on OTP submission (try 10 wrong codes back to
  back)? Is the OTP returned anywhere in the response/JS/API for "debugging"? Does
  the OTP endpoint respond differently for a right-length-wrong-value vs a
  wrong-length code (oracle)? Feed this endpoint to `jsonjuggler` if it's JSON —
  `{"otp": true}` / `{"otp": 0}` / `{"otp": null}` bypasses are common here.
- **Session handling**: does the session token actually invalidate on logout
  (replay the pre-logout cookie after logging out)? Does changing the account
  password invalidate other active sessions? Is the session cookie missing
  `HttpOnly`, `Secure`, or `SameSite`?

### 7b. Access control — IDOR & BAC (use both test accounts)
IDOR is the single most-paid web bug class — roughly 30% of all paid submissions
across programs. Work through all 10 variants, not just the obvious URL-swap one:

| Variant | What to test |
|---|---|
| Direct | `/api/users/123` → `/api/users/456` |
| Body param | `{"user_id": 456}` in a POST/PUT JSON body |
| GraphQL node | `{ node(id: "base64(OtherType:123)") { ... } }` |
| Batch/bulk | `/api/users?ids=1,2,3,4,5` — request multiple at once |
| Nested | parent ID swap: `/orgs/{org_id}/users/{user_id}` |
| File path | `/files/download?path=../other-user/file.pdf` |
| Predictable ID | sequential ints, timestamps, short UUIDs |
| Method swap | GET blocked (403)? Try PUT/PATCH/DELETE on the same path |
| Version rollback | v2 blocked? Try the same endpoint on `/api/v1/` |
| Header injection | `X-User-ID: victim_id`, `X-Org-ID: victim_org` |

Procedure: log in as A, note every object ID that appears in A's requests. Log in
as B, replay A's exact requests with A's IDs using B's session. **Full response
with A's data = IDOR**; a partial leak (name back but not email) still counts as
information disclosure at minimum. Repeat across every CRUD verb — write-access
IDOR (B can edit/delete A's object) is higher severity than read. Also check API
v1-vs-v2 differences, GraphQL `node()` queries bypassing per-type field auth, and
WebSocket messages carrying client-supplied IDs.

Admin/role-gated routes: with a low-privilege account, directly request every
admin-looking path found in JS/crawl (`/admin`, `/api/admin/*`, dashboard-only API
calls) — don't rely on the UI hiding the nav link; hit the endpoint directly.

**Chain it up**: IDOR+read PII = Medium. IDOR+write = High. IDOR+admin endpoint =
Critical (privilege escalation). IDOR+account-takeover path = Critical.

### 7c. Injection (using candidates from step 5's `gf` output)
- **SQLi**: for each `candidates_sqli.txt` entry, try a single quote `'` first and
  diff the response against baseline (error message, 500, different body length).
  If a diff appears, confirm with a boolean pair: `' AND 1=1-- -` vs `' AND 1=2-- -`
  — a real difference between the two confirms injection without needing `sqlmap`
  yet. Once confirmed manually, hand the exact request to
  `sqlmap -r request.txt --batch --level=3 --risk=2` for exploitation depth.
- **XSS**: for each `candidates_xss.txt` entry, identify the reflection context
  first (HTML body / attribute / JS string / URL) by sending a unique marker
  string, then pick the matching payload class for that context — don't throw
  generic `<script>alert(1)</script>` everywhere. Confirm execution in an actual
  browser (XSS firing in a raw curl response ≠ confirmed), or with
  `dalfox url <endpoint> --silence` for scale.
- **SSTI**: on any field that renders back into a page (name, bio, comment,
  template-driven emails), try `${{7*7}}`, `{{7*7}}`, `#{7*7}`, `<%= 7*7 %>` — a
  `49` in the output confirms template injection; escalate from there to the
  engine-specific RCE payload (identify engine first via error messages or by
  which syntax triggered).
- **SSRF**: any param/field that fetches a URL server-side (webhooks, avatar-by-URL,
  PDF export, SSO metadata URL, import-from-URL — "every import-from-URL feature
  has historically had SSRF") — point it at a Burp Collaborator URL first to
  confirm out-of-band callback, then pivot to cloud metadata once blind SSRF is
  confirmed. If the direct IP/hostname is filtered, cycle through these bypasses
  before giving up:

  | Bypass | Payload | Notes |
  |---|---|---|
  | Decimal IP | `http://2130706433/` | 127.0.0.1 as one decimal |
  | Hex IP | `http://0x7f000001/` | hex form |
  | Octal IP | `http://0177.0.0.1/` | octal 0177 = 127 |
  | Short IP | `http://127.1/` | abbreviated |
  | IPv6 loopback | `http://[::1]/` | — |
  | IPv6-mapped | `http://[::ffff:127.0.0.1]/` | IPv4-mapped IPv6 |
  | Redirect chain | `http://attacker.com/` → 302 → `http://169.254.169.254` | check every hop |
  | DNS rebinding | domain that resolves external first, internal on re-check | TOCTOU |
  | URL-encoding | `http://127.0.0.1%2523@attacker.com` | parser confusion |
  | Protocol smuggling | `gopher://127.0.0.1:6379/_INFO` | hits Redis/other internal protocols |

  Impact ladder: DNS-only callback = informational (don't submit alone) → internal
  service reachable = Medium → cloud metadata readable = High → metadata + exfil'd
  keys = Critical → Docker/K8s API reachable = Critical (near-direct RCE). Once
  metadata is readable:
  ```
  http://169.254.169.254/latest/meta-data/iam/security-credentials/        # AWS: role name
  http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>  # then the keys
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token  # GCP (needs header: Metadata-Flavor: Google)
  http://169.254.169.254/metadata/instance?api-version=2021-02-01          # Azure (needs header: Metadata: true)
  ```
- **Open redirect → OAuth chain**: alone it's a dead finding (always rejected) —
  only valuable if you can show it leads to OAuth code/token theft. If a redirect
  param is filtered, try:

  | Bypass | Payload |
  |---|---|
  | Double URL encoding | `%252F%252F` decodes to `//` after double-decode |
  | Backslash | `https://target.com\@evil.com` |
  | Missing protocol | `//evil.com` |
  | @-trick | `https://target.com@evil.com` |
  | Protocol-relative | `///evil.com` |
  | Whitespace injection | `//evil%09.com` |
  | Fragment trick | `https://evil.com#target.com` |
  | Null byte | `https://evil.com%00target.com` |
  | Parameter pollution | `?next=target.com&next=evil.com` (last value wins) |
  | Path confusion | `/redirect/..%2F..%2Fevil.com` |

  Then check whether the app's OAuth `redirect_uri` validation accepts your domain
  (wildcard, prefix match, or no validation at all) — if it does, the chain is
  redirect → auth-code interception → account takeover, which is report-worthy.

### 7d. Business logic
- Walk every multi-step flow (checkout, signup, invite, approval) and try
  skipping steps by calling a later-step API directly without completing earlier
  ones. Try replaying a completed step twice. Try going backward after completion.
- If there's pricing/quantity/discount logic: try negative numbers, zero, and
  floats where an int is expected; try applying the same coupon/discount code
  twice in parallel requests (race condition — see below).
- Race conditions: find any endpoint with a one-time effect (redeem coupon,
  claim reward, withdraw funds, like/vote once). Fire 10–20 identical requests
  in parallel (Burp's "Send group in parallel" / Turbo Intruder) and check if the
  effect happened more than once.

### 7e. File upload (if present)
Work through the bypass table rather than guessing randomly:

| Bypass | Technique |
|---|---|
| Double extension | `file.php.jpg`, `file.php%00.jpg` |
| Case variation | `file.pHp`, `file.PHP5` |
| Alternative extensions | `.phtml`, `.phar`, `.shtml`, `.inc` |
| Content-Type spoof | `image/jpeg` header with actual PHP content |
| Magic bytes | `GIF89a; <?php system($_GET['c']); ?>` |
| `.htaccess` upload | `AddType application/x-httpd-php .jpg` |
| SVG stored XSS | `<svg onload=alert(1)>` |
| Race condition | upload + request the file before cleanup/AV scan runs |
| Polyglot | a file that's simultaneously valid JPEG and valid PHP |
| Zip slip | `../../etc/cron.d/shell` as a path inside an uploaded archive |

Magic byte reference for spoofing content checks: JPEG `FF D8 FF`, PNG
`89 50 4E 47 0D 0A 1A 0A`, GIF `47 49 46 38`, PDF `25 50 44 46`, ZIP/DOCX/XLSX
`50 4B 03 04`.

Also check: is the upload path predictable/guessable, and is the uploaded file
served back with execution rights (upload a harmless marker file, then `curl` it
directly to see if it renders as a script vs. plain text)? Path traversal in the
filename field itself is also worth a quick try.

### 7f. CSRF
- For every state-changing request (password change, email change, fund transfer,
  settings update), check whether a CSRF token is present and actually validated
  — remove the token entirely, then resend with a blank/different value, and see
  if the request still succeeds. Also check if `SameSite` cookie attribute alone
  is relied on (bypassable in some cross-site navigation scenarios).

### 7g. Rate limiting
- Hit login, OTP, password reset, and any paid/limited-resource action 20–30
  times in quick succession. No throttling/lockout/captcha trigger = findable on
  its own, and makes brute-force/enumeration findings above more severe.

### 7h. Hand off to jsonjuggler
Any JSON-bodied endpoint touched in 7a (auth/OTP/reset) or 7c (search/filter
params) that does a server-side comparison gets a dedicated pass with the
`jsonjuggler` skill — type-juggling bugs in these exact endpoint types are easy
to walk past with the manual tests above since they only show up under
non-obvious payload types (`null`, `{}`, `{"$ne":""}`, etc.), not normal strings.

Use the `BugBountySkills` KB (if loaded in-session) as the deep reference for any
vuln class above once you have a lead — it has the exhaustive payload lists and
WAF-bypass variants this skill intentionally keeps lean.

## 8. Before you write anything up
Don't report bug A in isolation — map its siblings in the same controller/module
and test them for the same pattern first (IDOR → try PUT/DELETE on the same
endpoint; SSRF → check for cloud metadata reachability; XSS → check if the
session cookie lacks HttpOnly). Chained findings pay 3–10x a single bug, and most
programs read "one report per bug" submissions less favorably than one strong
chained report.

Once you have something concrete — chained or not — run it through the
**`huntgate`** skill before writing a single word of the report. It is not
optional polish: it's the gate that kills theoretical findings before you waste
an hour writing them up, and it's what turns a confirmed bug into a submission a
triager will actually pay.

## Output
Per-app folder: `map/sitemap.txt`, `map/js_endpoints.txt`, `map/candidates_*.txt`,
Burp project file with full traffic log, findings notes.
