# 🕸️ bb-skills

Five [Claude Code](https://claude.com/claude-code) / [OpenCode](https://opencode.ai) skills that turn
an agentic coding CLI into a structured bug bounty workflow — recon → triage → deep
app testing → gated-host fuzzing → JSON type-confusion auditing.

Each skill is a self-contained `SKILL.md` with real commands (subfinder, httpx, katana,
ffuf, nuclei, Burp workflow, etc.), not pseudocode. They're designed to be used in
order, with each skill's output feeding the next one's input.

```
╔══════════════════════════════════════════════════════════════════════════╗
║                      bb-skills — hunter pipeline                         ║
╚══════════════════════════════════════════════════════════════════════════╝

                        ┌──────────────────┐
                        │   reconforge     │  subdomain enum
                        │                  │  + DNS bruteforce
                        └────────┬─────────┘
                                 │ resolved subdomains
                                 ▼
                        ┌──────────────────┐
                        │   scopehawk      │  live probing
                        │                  │  + screenshots + tech-ID
                        └────────┬─────────┘
                                 │ prioritized targets
                      ┌──────────┴──────────┐
                      ▼                     ▼
              ┌───────────────┐     ┌───────────────┐
              │   TIER 1      │     │   TIER 2      │
              │  live apps    │     │  403/401/500  │
              └───────┬───────┘     └───────┬───────┘
                      ▼                     ▼
              ┌───────────────┐     ┌───────────────┐
              │   deepdive    │     │  wallbreaker  │
              │  map + test   │     │  bypass + ID  │
              └───────┬───────┘     └───────┬───────┘
                      ▲                     │
                      │   unblocked host    │
                      └─────────────────────┘
                      │
                      │ JSON auth / OTP / reset
                      │ endpoint found
                      ▼
              ┌───────────────┐
              │  jsonjuggler  │  type confusion
              │               │  / NoSQLi audit
              └───────────────┘
```

## Skills

| # | Skill | What it does |
|---|-------|---------------|
| 1 | [`reconforge`](./reconforge) | Passive subdomain aggregation (subfinder/amass/assetfinder/chaos/crt.sh), DNS bruteforce, permutation generation, resolution, and subdomain-takeover checks. |
| 2 | [`scopehawk`](./scopehawk) | Live-host probing (httpx), port sweep (naabu), screenshots (gowitness), tech fingerprinting, and grouping targets into Tier 1 (live apps) / Tier 2 (gated 403/500) / Tier 3 (noise). |
| 3 | [`deepdive`](./deepdive) | Prompts you to pick one live app from the Tier 1 list, then does full manual + automated mapping of it: Burp-logged browsing, two test accounts, JS file mining for hidden endpoints/secrets, historical URL mining, API spec discovery, and the actual vuln testing pass. |
| 4 | [`wallbreaker`](./wallbreaker) | Content/vhost fuzzing and 403-bypass technique sweep for gated hosts, plus 500-error triage and header/misconfig checks. Promotes unblocked hosts back to `deepdive`. |
| 5 | [`jsonjuggler`](./jsonjuggler) | Exhaustive JSON type-confusion / type-juggling / NoSQL-injection tester for any JSON API endpoint — 12-class payload matrix with a flatten → mutate → diff → confirm workflow. Called from within `deepdive` whenever a comparison-sensitive JSON endpoint shows up. Adapted from [0xSzit/json-juggler](https://github.com/0xSzit/json-juggler). |

## Order of execution

```
1 (reconforge) → 2 (scopehawk) ─┬→ 3 (deepdive) ──→ 5 (jsonjuggler)
                                 └→ 4 (wallbreaker) ──┘
                                    (wins loop back into 3)
```

- **1 → 2** run once per engagement (re-run periodically — scope changes over time).
- **3 and 4** run in parallel, once per host, across your Tier 1 / Tier 2 lists.
- **5** isn't a standalone phase — it's invoked from inside **3** whenever a JSON
  endpoint worth attacking (auth, OTP, token/reset compare, search/filter) turns up.
- Anything **4** unblocks gets promoted into Tier 1 and re-enters **3**.

## Install

### Claude Code
```bash
git clone https://github.com/<you>/bb-skills.git
cp -r bb-skills/*/ ~/.claude/skills/        # user-level, all projects
# or, per-project:
cp -r bb-skills/*/ ./.claude/skills/
```
Claude Code auto-discovers any folder containing a `SKILL.md` under `.claude/skills/`.
Just describe what you're doing in plain language ("run recon on target.com", "fuzz
this 403") and the matching skill activates automatically.

### OpenCode
```bash
git clone https://github.com/<you>/bb-skills.git
cp -r bb-skills/*/ ~/.opencode/skill/
```
(Check your OpenCode version's exact skills path — `opencode skill list` will confirm
discovery.)

## Requirements

These skills assume a standard recon/hunting toolbox is installed and on `$PATH`:

```
subfinder · amass · assetfinder · chaos · puredns · alterx · gotator · subzy
httpx · naabu · gowitness · whatweb · nuclei
katana · hakrawler · gau · waybackurls · gf · ffuf · feroxbuster
Burp Suite (manual proxy-logged testing)
```

Most are installable via `go install` ([ProjectDiscovery](https://github.com/projectdiscovery) tools)
or your package manager. `nuclei -ut` to update templates before first use.

## ⚠️ Scope & Authorization

Everything here is for **authorized testing only** — targets you own, or an active
bug bounty program / signed pentest engagement where the techniques used (DNS
bruteforce, fuzzing, injection payloads, write-method testing) are within the
program's rules of engagement. Passive recon is read-only and lower-risk; everything
from step 2 onward is active traffic against the target. Always check program scope
before running anything. Unauthorized testing is illegal.

## Credits

Built on patterns and references from:
- [0xSzit/BountyCompass](https://github.com/0xSzit/BountyCompass)
- [0xN0RMXL/BugBountySkills](https://github.com/0xN0RMXL/BugBountySkills)
- [0xSzit/json-juggler](https://github.com/0xSzit/json-juggler)

## License

MIT — use responsibly.
