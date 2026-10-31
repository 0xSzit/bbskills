---
name: huntgate
description: Use before writing up ANY finding from deepdive or wallbreaker — triggered by "is this worth reporting", "validate this finding", "write a report", "should I submit this". Kills weak/theoretical findings before time is wasted, then produces a submission-ready report. This is a mandatory checkpoint, not optional polish.
---

# Validate & Report

Never write a report before running this. A rejected report costs reputation and
validity-ratio score on every platform — this skill exists to stop that before it
happens.

## 0. The only question that matters
> **Can an attacker do this RIGHT NOW against a real user who took no unusual
> action — and does it cause real harm (stolen money, leaked PII, account
> takeover, code execution)?** If no → stop, don't write it up, move on.

## 1. Kill list — if the finding matches any of these, drop it (or go find the chain)

| Pattern | Why it dies |
|---|---|
| "Could theoretically allow..." | Not exploitable = not a bug |
| "An attacker with X, Y, Z conditions could..." | Too many preconditions |
| Wrong implementation, no practical impact | Harmless ≠ a bug |
| Dead code containing the bug | Not reachable |
| Source maps with no secrets in them | No impact |
| SSRF with DNS-callback only, no internal access/exfil | Not enough on its own |
| Open redirect alone | Needs an ATO or OAuth chain to matter |
| Missing CSP/HSTS/security headers alone | Always rejected |
| Missing SPF/DKIM/DMARC alone | Always rejected |
| GraphQL introspection enabled alone | Informational only |
| Banner/version disclosure with no working exploit | Always rejected |
| Clickjacking on a non-sensitive page | Always rejected |
| CORS wildcard with no credentialed-exfil PoC | Always rejected |
| Logout CSRF | Always rejected |
| Self-XSS | Always rejected |
| Rate-limit missing on a non-critical form | Always rejected |
| Session not invalidated on logout, alone | Always rejected |
| Missing HttpOnly/Secure cookie flag, alone | Always rejected |
| Host header injection, alone | Needs password-reset-poisoning chain |

**But a "dead" low finding is often half of a real bug — check the chain table
before you actually drop it:**

| Low finding | + chain with | = valid bug |
|---|---|---|
| Open redirect | OAuth `redirect_uri` accepts your domain | ATO via code theft |
| Clickjacking | a genuinely sensitive action + working PoC | account action hijack |
| CORS wildcard | credentials:include + real data returned | data theft |
| CSRF | a sensitive state-changing action | account takeover |
| No rate limit | OTP/password brute force | ATO |
| SSRF (DNS-only) | proof of internal network access | internal access disclosure |
| Host header injection | password-reset-link poisoning | ATO |
| Self-XSS | a login-CSRF to plant it on the victim | stored XSS on a real victim |
| S3 bucket listing | OAuth client_secret found in JS bundles inside it | OAuth compromise chain |

If you can build the right-hand side, it's a real finding — go get the second half
before writing anything.

## 2. A→B→C cluster hunting (do this before validating — chains pay 3–10x more)
When you confirm bug A, don't write it up yet — hunt the sibling first:

1. **Confirm A** with an actual HTTP request/response.
2. **Map siblings** — every endpoint in the same controller/module/API group.
3. **Test siblings** for the same bug pattern, and for different bug classes.
4. **Chain** — if a sibling has a different bug class, try combining.
5. **Quantify** — "affects N users" / "exposes $X value" / "N records."
6. **Then** validate and report — one report per chain, not per bug.

Known high-value chains:
```
IDOR (read)            -> PUT/DELETE on same endpoint        -> full data manipulation
SSRF (any)              -> cloud metadata 169.254.169.254     -> IAM key exfil -> RCE
XSS (stored)             -> HttpOnly not set on session cookie -> session hijack -> ATO
Open redirect            -> OAuth redirect_uri accepts your domain -> auth code theft -> ATO
S3 bucket listing        -> JS bundles contain OAuth secrets  -> OAuth chain
Rate limit bypass        -> OTP brute force                   -> account takeover
GraphQL introspection    -> missing field-level auth           -> mass PII exfil
Debug endpoint           -> leaked env vars                    -> cloud credential access
CORS reflects origin     -> tested with credentials:include    -> credentialed data theft
Host header injection    -> password reset poisoning           -> ATO via reset link
```

## 3. The 7-Question Gate — all 7 must be YES, any NO = stop
1. **Can I exploit this RIGHT NOW with a real PoC?** Write the exact HTTP
   request. Can't produce one → kill it.
2. **Does it affect a real user who took no unusual action?** No "the victim
   would need to..." with five preconditions.
3. **Is the impact concrete** (money, PII, ATO, RCE)? "Technically possible" is
   not impact. "I read the victim's SSN" is impact.
4. **Is this in scope** per the program's exact scope page — this host/endpoint,
   not just the root domain?
5. **Did I check for duplicates** — searched the program's Hacktivity/disclosed
   reports and recent changelog for this exact bug?
6. **Is this NOT on the kill list above** (or have I built the chain that makes
   it valid)?
7. **Would a tired triager reading this at 5pm on a Friday say "yes, real bug"?**

## 4. Pre-submission gates (run in order, ~20 min total)

**Gate 0 — Reality check (30s):** bug confirmed with real requests (not just code
reading), confirmed in-scope, reproducible from scratch, evidence captured
(screenshot/response/video).

**Gate 1 — Impact (2 min):** you can answer "what can an attacker DO that they
couldn't before?" in one sentence, and the answer is more than "see non-sensitive
data" — there's a real victim (another user, the company, financial loss).

**Gate 2 — Deduplication (5 min):** searched Hacktivity/Crowdstream for this
program + similar title, searched the target's public GitHub issues, read the
5 most recent disclosed reports for this program, confirmed this isn't a documented
known issue.

**Gate 3 — Report quality (10 min):** title states vuln class + location +
impact in one sentence; steps to reproduce are a copy-pasteable HTTP request;
evidence shows actual impact, not just a 200 response; severity matches both
CVSS and the program's own severity definitions; remediation is 1–2 concrete
sentences.

## 5. CVSS 3.1 quick reference

| Bug class | Typical CVSS | Severity |
|---|---|---|
| IDOR (read PII) | 6.5 | Medium |
| IDOR (write/delete) | 7.5 | High |
| Auth bypass → admin | 9.8 | Critical |
| Stored XSS | 5.4–8.8 | Med–High |
| SQLi (data exfil) | 8.6 | High |
| SSRF (cloud metadata) | 9.1 | Critical |
| Race condition (double spend) | 7.5 | High |
| GraphQL auth bypass | 8.7 | High |
| JWT `alg: none` | 9.1 | Critical |

## 6. Report template (adapt per platform)

```
Title: [Vuln Class] in [exact endpoint/feature] allows [attacker role] to [impact]

## Summary
2–3 sentences: what it is, where it is, what the attacker can do.

## Steps To Reproduce
1. Log in as attacker (account A)
2. Send request: [paste exact HTTP request]
3. Observe: [exact response showing the bug]
4. Confirm: [what the attacker gained]

## Supporting Material
[Screenshot / video / Burp request-response pair]

## Impact
An attacker can [specific action] resulting in [specific harm].
Quantify if possible: "affects all N users" / "exposes X records."

## Severity Assessment
CVSS 3.1: X.X ([label]) — Attack Vector: Network | Privileges: None | UI: None

## Fix Suggestion
One or two concrete sentences, not a lecture.
```

Good title formula: `[Bug Class] in [exact endpoint] allows [attacker role] to
[impact] [victim scope]` — e.g. "IDOR in /api/v2/invoices/{id} allows
authenticated user to read any customer's invoice data." Bad: "IDOR
vulnerability found," "Security issue in API."

Writing rules: start sentences with the impact, not the vuln name. Write like
you're explaining to a smart developer, not a textbook ("I found that..." not "A
vulnerability was discovered..."). One concrete example beats three abstract
sentences. Keep it under 600 words — triagers skim long reports.

## 7. Final 60-second checklist before hitting submit
```
[ ] Title follows the formula above
[ ] First sentence states the exact impact in plain English
[ ] Steps to Reproduce is a copy-paste-ready HTTP request
[ ] Response/screenshot showing the bug is attached
[ ] Two test accounts were used (not one account testing itself) where relevant
[ ] CVSS score calculated and included
[ ] Severity claimed matches the impact described — don't overclaim
[ ] No typos in the endpoint path or parameter names
```

## 8. If a triager pushes back
| They say | You counter with |
|---|---|
| "Requires authentication" | "Attacker only needs a free account, no special role" |
| "Limited impact" | "Affects N users / this PII type / $X value" |
| "Already known" | "Show me the report number — I searched and found none" |
| "By design" | "Show me documentation stating this is intended" |
| "Low CVSS score" | "CVSS doesn't capture business impact — attacker can steal X" |

Sources: methodology and tables adapted from shuvonsec/claude-bug-bounty and
awarexone/Agentic-Bug-Hunter.
