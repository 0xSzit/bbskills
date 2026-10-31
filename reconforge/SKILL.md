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

## Output
`resolved/final_resolved.txt` — the canonical list. Hand this to the
`live-host-fingerprint` skill next.

## Also worth doing (don't skip)
- Wayback/gau URL collection and ASN/IP-range discovery can surface subdomains CT logs miss:
  `echo $TARGET | waybackurls`, `asnmap -d $TARGET`.
- Re-run this whole skill periodically on long engagements — programs add assets over time.
