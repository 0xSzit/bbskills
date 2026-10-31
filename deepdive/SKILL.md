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

## 7. Now actually test
With the full map + two accounts, work through: auth flows, IDOR (swap IDs/tokens
between account A and B), access control on admin routes, injection points found in
step 5, business-logic abuse, file upload, CSRF, rate limiting. This is where the
`BugBountySkills` vuln-class references (XSS/SQLi/SSRF/IDOR/JWT/etc.) are the right
companion material to pull in per finding.

For any JSON-bodied endpoint doing auth, OTP/token verification, password reset,
or search/filter comparisons, run the `jsonjuggler` skill against it —
these bugs hide in loose-comparison and NoSQL-operator handling and are easy to miss
with normal payload-based fuzzing.

## Output
Per-app folder: `map/sitemap.txt`, `map/js_endpoints.txt`, `map/candidates_*.txt`,
Burp project file with full traffic log, findings notes.
