---
name: wallbreaker
description: Use for Tier-2 "gated" hosts — subdomains returning 403, 401, 500, 502, or other non-standard responses that aren't full browsable apps. Triggered by "fuzz this", "bypass this 403", "what's behind this host", or "check this error page". Covers content/vhost fuzzing, header manipulation, and 403-bypass techniques.
---

# Fuzzing & Triage for 403 / 500 / Gated Hosts

Input: `tier2_gated.txt` from live-host-fingerprint skill. Work one host at a time.

## 1. Content discovery (what's actually there)
```bash
ffuf -u https://host.target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt \
  -mc 200,201,204,301,302,307,401,403,405,500 \
  -fc 404 -t 50 -o fuzz/content.json

feroxcorder -u https://host.target.com -w raft-medium-words.txt --silent -x php,json,bak,old,env
```
Check for backup/debug/config files on every gated host:
`.git/HEAD`, `.env`, `.env.bak`, `web.config`, `/actuator`, `/debug`, `/server-status`,
`config.php.bak`, `docker-compose.yml`.

## 2. 403 bypass techniques
Try systematically; a single hit often unblocks the whole directory:
```bash
# Path manipulation
curl -s https://host.target.com/admin/..;/
curl -s https://host.target.com/%2e/admin
curl -s https://host.target.com/admin/.
curl -s https://host.target.com//admin//
curl -s https://host.target.com/admin%20
curl -s https://host.target.com/admin%09

# Header manipulation (bypass via trusted-IP / internal-only logic)
curl -s https://host.target.com/admin -H "X-Forwarded-For: 127.0.0.1"
curl -s https://host.target.com/admin -H "X-Forwarded-For: localhost"
curl -s https://host.target.com/admin -H "X-Originating-IP: 127.0.0.1"
curl -s https://host.target.com/admin -H "X-Custom-IP-Authorization: 127.0.0.1"

# Method override
curl -s -X POST https://host.target.com/admin
curl -s -X GET https://host.target.com/admin -H "X-HTTP-Method-Override: GET"

# Or run a tool that tries all of these at once
python3 403bypasser.py -u https://host.target.com/admin
nuclei -u https://host.target.com/admin -tags 403-bypass
```

## 3. vhost / virtual-host fuzzing
Some apps are only reachable via Host header, not the resolved subdomain path:
```bash
ffuf -u https://TARGET_IP -H "Host: FUZZ.target.com" \
  -w subdomains-top1million.txt -fs <baseline_size> -o fuzz/vhost.json
```

## 4. 500 / error page triage
- Read the full error body — stack traces often leak framework version, internal
  paths, SQL fragments, or debug mode being on (immediately report-worthy on its own
  in many programs if it's verbose/debug info disclosure).
- Try to make it 500 in a *different* way (malformed JSON, oversized input, wrong
  content-type) to see if it's consistently misconfigured or one-off.
- Check response headers always: `Server`, `X-Powered-By`, `X-AspNet-Version`,
  cache headers, CSP, `Via`/`X-Cache` (reveals CDN/proxy chain — useful for SSRF and
  cache-poisoning follow-up).

## 5. Headers & misc checks on every gated host
```bash
curl -sI https://host.target.com
nuclei -u https://host.target.com -tags misconfig,headers,exposure,cve -o fuzz/nuclei_misc.txt
```

## 6. Search the internet for this host
Google/GitHub dork and search engines for leftover references:
```
site:host.target.com
"host.target.com" site:github.com
"host.target.com" filetype:env OR filetype:json OR filetype:log
```
Check Wayback/archive for this exact host too — old snapshots sometimes show
pre-auth content that's now gated, revealing the app behind the 403.

## Output
Anything that unblocks a 403, or any 500 leaking internals, gets promoted to
`tier1_apps.txt` and handed to the `app-mapping-testing` skill.
