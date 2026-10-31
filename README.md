# 🕸️ bb-skills

Six [Claude Code](https://claude.com/claude-code) / [OpenCode](https://opencode.ai) skills that turn
an agentic coding CLI into a structured bug bounty workflow — recon → triage → deep
app testing → gated-host fuzzing → JSON type-confusion auditing → validation & reporting.

Each skill is a self-contained `SKILL.md` with real commands (subfinder, httpx, katana,
ffuf, nuclei, Burp workflow, etc.) and real testing procedures (exact bypass tables,
IDOR variants, CVSS references), not pseudocode or "go test for XSS." They're designed
to be used in order, with each skill's output feeding the next one's input.

```
╔══════════════════════════════════════════════════════════════════════════╗
║                      bb-skills — hunter pipeline                         ║
╚══════════════════════════════════════════════════════════════════════════╝

                        ┌──────────────────┐
                        │   reconforge     │  subdomain enum + DNS
                        │                  │  bruteforce + cloud/secrets
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
              └───────┬───────┘
                      │  confirmed finding
                      ▼
              ┌───────────────┐
              │   huntgate    │  7-Question Gate
              │               │  + chain hunting + report
              └───────────────┘
```

## Skills

| # | Skill | What it does |
|---|-------|---------------|
| 1 | [`reconforge`](./reconforge) | Passive subdomain aggregation, DNS bruteforce, permutation generation, resolution, subdomain-takeover checks, cloud/S3 bucket enumeration, GitHub secrets dorking, and a quick-wins checklist run across every resolved host. |
| 2 | [`scopehawk`](./scopehawk) | Live-host probing, port sweep, screenshots, tech fingerprinting, and grouping targets into Tier 1 (live apps) / Tier 2 (gated 403/500) / Tier 3 (noise). |
| 3 | [`deepdive`](./deepdive) | Prompts you to pick one live app from the Tier 1 list, then does full manual + automated mapping: Burp-logged browsing, two test accounts with captured auth state for A/B diffing, JS mining, historical URL mining, API spec discovery, and per-category testing procedures (auth, IDOR's 10 variants, injection, business logic/race conditions, file upload bypass table, CSRF, rate limiting) with exact bypass tables for SSRF and open redirect. |
| 4 | [`wallbreaker`](./wallbreaker) | Content/vhost fuzzing, a real 403-bypass technique checklist, and 500-error triage that greps responses for framework-specific stack-trace patterns instead of just reading them by eye. Promotes unblocked hosts back to `deepdive`. |
| 5 | [`jsonjuggler`](./jsonjuggler) | Exhaustive JSON type-confusion / type-juggling / NoSQL-injection tester — 12-class payload matrix with a flatten → mutate → diff → confirm workflow. Called from within `deepdive` whenever a comparison-sensitive JSON endpoint shows up. |
| 6 | [`huntgate`](./huntgate) | The mandatory checkpoint before writing up anything: a kill-list for theoretical findings, an A→B→C chain-hunting protocol (chains pay 3–10x more than single bugs), the 7-Question Gate, CVSS reference, and ready-to-fill report templates. |

## Order of execution

```
1 (reconforge) → 2 (scopehawk) ─┬→ 3 (deepdive) ──┬→ 5 (jsonjuggler) ─┐
                                 └→ 4 (wallbreaker)─┘                  ├→ 6 (huntgate)
                                    (wins loop back into 3)  ──────────┘
```

- **1 → 2** run once per engagement (re-run periodically — scope changes over time).
- **3 and 4** run in parallel, once per host, across your Tier 1 / Tier 2 lists.
- **5** isn't a standalone phase — it's invoked from inside **3** whenever a JSON
  endpoint worth attacking (auth, OTP, token/reset compare, search/filter) turns up.
- Anything **4** unblocks gets promoted into Tier 1 and re-enters **3**.
- **6 runs before every single report**, no exceptions — including findings from
  `wallbreaker` that look report-worthy on their own, not just ones that came out
  of a full `deepdive` pass.

## Hunting rules (apply across every skill)

These aren't per-skill advice — they govern how you use the whole pipeline:

1. **Read full scope first.** Verify every asset is actually owned by the target
   org before touching it, every time — not just once at the start.
2. **No theoretical bugs.** "Can an attacker do this right now against a real
   user who took no unusual action?" If no, stop — don't write it up.
3. **5-minute rule.** A target showing nothing but 401/403/404 after 5 minutes
   of probing — move on, don't spiral.
4. **One-hour rule.** Stuck on one target an hour with no progress — switch
   context, come back later or not at all.
5. **Impact-first hunting.** Before diving into a target, ask "what's the worst
   thing if auth were broken here?" If the answer is nothing valuable, skip it.
6. **Hunt less-saturated classes.** XSS/SSRF/XXE have the most competition —
   business logic, race conditions, OAuth/OIDC chains, and cache poisoning pay
   well with less competition.
7. **Chain before you report.** See `huntgate`'s A→B→C protocol — a confirmed
   bug is a reason to look at its siblings before writing anything up, not a
   reason to stop.
8. **Validate before you write.** `huntgate` is not optional polish — run it
   before every report, not after you've already spent 30 minutes drafting one.

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
this 403", "validate this finding") and the matching skill activates automatically.

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
trufflehog · gitleaks
Burp Suite (manual proxy-logged testing, Turbo Intruder for race conditions)
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

Built on patterns, tables, and methodology from:
- [0xSzit/BountyCompass](https://github.com/0xSzit/BountyCompass)
- [0xN0RMXL/BugBountySkills](https://github.com/0xN0RMXL/BugBountySkills)
- [0xSzit/json-juggler](https://github.com/0xSzit/json-juggler)
- [shuvonsec/claude-bug-bounty](https://skillvault.md/shuvonsec/claude-bug-bounty/bug-bounty) — 7-Question Gate, A→B chaining, bypass tables, CVSS/reporting templates
- [awarexone/Agentic-Bug-Hunter](https://github.com/awarexone/Agentic-Bug-Hunter) — auth-aware hunting, hunting rules
- [instavm/security-skills](https://github.com/instavm/security-skills) — MITM auth-diff pattern for IDOR detection

## License

MIT — use responsibly.
