---
name: reconforge
description: Use when starting a bug bounty engagement on a new target/domain, or when asked to "run recon", "enumerate subdomains", "find assets", or "DNS bruteforce" a domain. Produces a deduplicated, resolved subdomain list ready for live-host probing. Only run against domains the user confirms are in-scope for an authorized program.
---

# Recon & Subdomain Enumeration

Scope gate first: confirm the root domain(s) are in an authorized bug bounty / pentest
scope before running anything. Passive steps (CT logs, archives) are low-risk; DNS
bruteforce and permutation resolution are active — respect program rate limits.

Work in a per-target dir: `mkdir -p recon/<target>/{raw,resolved,logs} && cd recon/<target>`

## 1. Passive aggregation (no traffic to target infra)
Run all in parallel, write to `raw/`:
```bash
subfinder -d $TARGET -all -recursive -o raw/subfinder.txt
amass enum -passive -d $TARGET -o raw/amass.txt
assetfinder --subs-only $TARGET > raw/assetfinder.txt
chaos -d $TARGET -o raw/chaos.txt                      # needs CHAOS_KEY
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" | jq -r '.[].name_value' \
  | sed 's/\*\.//' > raw/crtsh.txt
github-subdomains -d $TARGET -o raw/gh_subs.txt         # needs GITHUB_TOKEN
```

Merge and normalize:
```bash
cat raw/*.txt | sed 's/^\*\.//; s/\.$//' \
  | grep -E "^[a-zA-Z0-9._-]+\.${TARGET//./\\.}$" \
  | tr '[:upper:]' '[:lower:]' | sort -u > resolved/seed.txt
```

## 2. DNS bruteforce (active)
Decision: seed list ≥200 → skip straight brute, go to permutation (step 3).
Seed <50 → brute force first.
```bash
wget -q https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt -O resolvers.txt
puredns bruteforce /usr/share/seclists/Discovery/DNS/n0kovo_subdomains.txt $TARGET \
  --resolvers resolvers.txt --rate-limit 5000 \
  --wildcard-tests 30 --wildcard-batch 1500000 \
  --write resolved/bruteforce.txt
```

## 3. Permutation generation (from known subs → find siblings)
```bash
alterx -l resolved/seed.txt -enrich -o raw/permutations.txt
gotator -sub resolved/seed.txt -perm raw/permutations.txt -depth 1 -numbers 5 -mindup -adv -md > raw/altered.txt
cat raw/permutations.txt raw/altered.txt | sort -u > raw/all_perm.txt
puredns resolve raw/all_perm.txt --resolvers resolvers.txt --rate-limit 5000 \
  --write resolved/permutation_resolved.txt
```

## 4. Resolve everything, dedupe
```bash
cat resolved/seed.txt resolved/bruteforce.txt resolved/permutation_resolved.txt \
  | sort -u > resolved/all_subs.txt
puredns resolve resolved/all_subs.txt --resolvers resolvers.txt \
  --write resolved/final_resolved.txt
```

## 5. Takeover check (cheap, do it now while you have the list)
```bash
subzy run --targets resolved/final_resolved.txt --hide_fails --verify_ssl
# or: nuclei -l resolved/final_resolved.txt -tags takeover
```

## 6. Wildcard DNS sanity check
If `*.target.com` resolves to a single IP for garbage subdomains, you have a wildcard —
filter it out before trusting bruteforce hits (`puredns` flags already handle most of this,
but spot-check with `dig random123456.target.com`).

## 7. Cloud asset enumeration
Buckets and storage accounts don't show up in DNS recon — brute them separately:
```bash
for suffix in dev staging test backup api data assets static cdn prod uploads; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "https://${TARGET}-${suffix}.s3.amazonaws.com/")
  [ "$code" != "404" ] && echo "$code  ${TARGET}-${suffix}.s3.amazonaws.com"
done
# Firebase open-read check
curl -s "https://${TARGET}.firebaseio.com/.json"
```
A listable bucket is low severity alone — but check what's inside: JS bundles in
S3 buckets have leaked OAuth `client_secret` values in real paid reports (see the
`huntgate` skill's chain table). Don't stop at "bucket is public," open it.

## 8. GitHub/secrets recon (if the target has public repos)
```bash
# github-subdomains already pulled subs in step 1; now hunt for leaked creds
trufflehog github --org=$TARGET_ORG --json > raw/trufflehog.json
gitleaks detect --source=. --report-path raw/gitleaks.json   # on any cloned target repo
# Manual dorking on github.com search UI (no good CLI for this):
#   "target.com" filetype:env OR filetype:json OR filetype:log
#   "target.com" password OR secret OR api_key
```

## 9. Quick wins checklist — run these against every resolved host, not just Tier 1
```
[ ] Subdomain takeover (step 5 above)
[ ] Exposed .git (/.git/config, /.git/HEAD)
[ ] Exposed env files (/.env, /.env.local, /.env.bak)
[ ] Default credentials on any admin panel surfaced
[ ] GraphQL introspection enabled (informational alone — note it, keep moving)
[ ] Spring actuators (/actuator/env, /actuator/heapdump)
```
Framework-specific quick wins once tech is fingerprinted (full fingerprinting
happens in `scopehawk`, but these are cheap to check immediately):
```
Laravel:   /horizon  /telescope  /.env  /storage/logs/laravel.log
WordPress: /wp-json/wp/v2/users  /xmlrpc.php  /?author=1
Node.js:   /.env  /graphql (try introspection)  /_debug
```

## Output
`resolved/final_resolved.txt` — the canonical list. Hand this to the
`scopehawk` skill next.

## Also worth doing (don't skip)
- Wayback/gau URL collection and ASN/IP-range discovery can surface subdomains CT logs miss:
  `echo $TARGET | waybackurls`, `asnmap -d $TARGET`.
- Re-run this whole skill periodically on long engagements — programs add assets over time.
- **5-minute rule**: if a specific asset shows nothing but 401/403/404 after 5
  minutes of probing, move on — don't spiral on a dead target this early.
