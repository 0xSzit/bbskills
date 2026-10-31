---
name: scopehawk
description: Use after subdomain enumeration is done, when asked to "probe live hosts", "screenshot subdomains", "fingerprint tech stack", or "prioritize targets". Takes a resolved subdomain list and produces a grouped, prioritized worklist of live applications with tech stack and screenshots.
---

# Live Host Discovery, Visual Recon, Fingerprinting & Prioritization

Input: `resolved/final_resolved.txt` from the reconforge skill.

## 1. Live host probing
```bash
cat resolved/final_resolved.txt | httpx -silent -threads 100 \
  -status-code -title -tech-detect -location -content-length -cl -ip -cname \
  -follow-redirects -o live/httpx_full.txt -json -o live/httpx.json
```
Also probe common non-80/443 ports if scope allows:
```bash
naabu -list resolved/final_resolved.txt -top-ports 1000 -o live/open_ports.txt
httpx -l live/open_ports.txt -silent -o live/httpx_extra_ports.txt
```

## 2. Visual recon (screenshots)
```bash
gowitness scan file -f resolved/final_resolved.txt --threads 10 -P live/screenshots/
# or: aquatone
```
Screenshots are how you fast-triage hundreds of hosts — skim for login portals,
admin panels, default install pages, staging banners, error pages revealing stack traces.

## 3. Tech stack fingerprinting
`httpx -tech-detect` already gives Wappalyzer-style hints; deepen with:
```bash
whatweb -i resolved/final_resolved.txt --log-json=live/whatweb.json
nuclei -l resolved/final_resolved.txt -tags tech,panel,exposure -o live/nuclei_fingerprint.txt
```
Cloud/CDN identification matters for scope and SSRF targets later:
```bash
cat live/httpx.json | jq -r 'select(.cdn==true) | .input' > live/behind_cdn.txt
```

## 4. Group & prioritize
Parse `live/httpx.json` and bucket into:

- **Tier 1 — active full-app targets**: 200/301/302 with real content, login forms,
  API responses, distinct tech stack (these feed `deepdive` skill).
- **Tier 2 — interesting but gated**: 401/403/404-with-custom-body, WAF-fronted,
  unusual headers (these feed `wallbreaker` skill).
- **Tier 3 — noise**: parked domains, identical default vhost pages, CDN-only with
  no distinguishing content, dead hosts (deprioritize, revisit later).

De-duplicate by response fingerprint so you don't waste time on 50 subdomains that
all serve the same default page:
```bash
cat live/httpx.json | jq -r '[.input,.status_code,.content_length,.hash.body_mmh3] | @tsv' \
  | sort -u -k4,4 > live/unique_by_hash.txt
```

## Output
Three worklists: `tier1_apps.txt`, `tier2_gated.txt`, `tier3_noise.txt` (plus
screenshots + tech-detect data) — ready for the next two skills.
