# How short can a case-insensitive link token be and stay safe?

Research for issue #2 (map #1). Feeds the decision in grilling ticket #8; nothing here is decided.
Vocabulary: **Invitation Token**, **Share Token**, **Invitation**, **Open Questionnaire**, **Server Draft** as in `GLOSSARY.md`.

## TL;DR

- Today's link is a reversible HashID of a sequential id. That is an encoding, not a secret: anyone who learns the HashID salt, or enumerates ids, reaches every Invitation. The W3C TAG calls out sequential identifiers as easy to enumerate and asks for a secure random generator [W3C-CAP].
- A capability link's safety depends on **bits of randomness × how fast anyone can guess × how many tokens are live at once**. The expected time to the first hit is `2^bits / (guesses per second × live tokens)`. OWASP's session-ID example uses this model [OWASP-SM].
- Primary sources set three different floors: **64 bits** for session IDs (OWASP Session Management Cheat Sheet [OWASP-SM]; NIST SP 800-63B-4 §5.1 session secrets [NIST-63B]), **120 bits** for capability URLs (W3C TAG [W3C-CAP]), and **128 bits** for reference session tokens (OWASP ASVS 5.0 V7.2.3 [ASVS-7]). None of them targets links that people retype. 128 bits is 26 Crockford characters, which is too long to retype.
- **Crockford Base32** fits "case-insensitive and retypable": 32 symbols at 5 bits each, no I/L/O/U, decoding folds case and maps O→0 and I/L→1, and hyphens are ignored [CROCKFORD].
- **Options:** 12 chars = 60 bits, 13 = 65, 15 = 75 (shown as `XXXXX-XXXXX-XXXXX`), 26 = 130.
- **Recommendation for #8: 15 characters (75 bits), in three hyphenated groups of five.** Take 1,000,000 live tokens and an attacker at OWASP's assumed 10,000 guesses/s with no throttling at all: the expected time to the first hit is about 120,000 years. That meets the OWASP 64-bit floor with a wide margin even if rate limiting fails. If #8 wants it shorter, **13 characters (65 bits)** is the shortest length that still meets the 64-bit floor, and it depends on rate limiting working.
- The leaks that matter more than length are logs (web server, API, Sentry), browser history on shared devices, the `Referer` header, third-party scripts (the bot check) and mail scanners that open links ahead of the person. The mitigations: `Referrer-Policy: no-referrer`, scrub tokens from logs and Sentry, expire tokens when the Questionnaire closes or the Invitation is submitted, never let a GET change state, and rate-limit the **API** lookup [W3C-CAP] [OWASP-FP].

## 1. What the primary sources require

| Source | Applies to | Requirement |
|---|---|---|
| OWASP Session Management Cheat Sheet [OWASP-SM] | session IDs | "at least 64 bits of entropy"; CSPRNG. Worked example: 64 bits, 10,000 guesses/s and 100,000 valid sessions give about 585 years to guess one. |
| NIST SP 800-63B-4 §5.1 [NIST-63B] | session secrets | at least 64 bits, from an approved random bit generator (§3.2.12). |
| NIST SP 800-63B-4 §3.1.2 / §3.2.2 [NIST-63B] | look-up secrets (recovery codes) | at least 6 decimal digits "or equivalent" (about 20 bits), only because the verifier caps consecutive failures at 100 per account. Secrets under 112 bits are stored salted and hashed. |
| W3C TAG, *Good Practices for Capability URLs* [W3C-CAP] | capability URLs | "120 bits or more" of entropy, secure random generator, HTTPS, expiry, revocation, rate limiting, Referrer policy, no third-party scripts, a GET must not trigger actions. Advisory, not normative. |
| OWASP ASVS 5.0 V7.2.3 (L1) [ASVS-7] | reference session tokens | CSPRNG and "at least 128 bits of entropy". |
| OWASP Forgot Password Cheat Sheet [OWASP-FP] | tokens sent in URLs | CSPRNG; "long enough to protect against brute-force attacks"; single use; expire; `Referrer-Policy: no-referrer`; protect against brute force, e.g. with rate limiting. |

**Reading:** NIST's 20-bit look-up secrets show how far rate limiting can lower the floor. They work only because the limit is **per account**. A link token has no account to lock: every guess is a fresh lookup against all live tokens at once. The per-account trick does not carry over, so the floor stays around 64 bits. The 120- and 128-bit figures assume no rate limiting and tokens that only machines handle. Going below them is a conscious deviation, and #8 should record it. ASVS V7.2.3 is the requirement a security audit is most likely to cite.

## 2. Alphabet: Crockford Base32

From the spec [CROCKFORD]:

- Symbols `0-9 A-H J K M N P-T V-Z` (32 symbols, 5 bits each). The spec excludes I and L (confusable with 1), O (confusable with 0) and U (accidental obscenity).
- Decoding is case-insensitive. O/o decodes as 0, I/i/L/l as 1. Encoding emits uppercase.
- Hyphens may be inserted for readability and are ignored on decode.
- Optional check symbol: the value mod 37, using five extra symbols `* ~ $ = U`.

Notes for this app:

- **Normalise before lookup:** uppercase, strip hyphens, map O→0 and I/L→1, then reject anything outside the alphabet. Do this in the SPA *and* in the API. RFC 3986 URL paths are case-sensitive, so the app has to provide case-insensitivity itself.
- **Hyphens** are unreserved characters in a URL path, so `form.luppa.app/7KQ3M-9XD2H-TAV4P` needs no escaping.
- **Crockford's mod-37 check symbol is URL-awkward.** `*`, `$` and `=` are reserved characters in URLs and hard to read aloud. A check character adds **no** security. Its only value is UX: the SPA can say "looks like a typo" without spending a server lookup, which would also count against the person's rate limit. If #8 wants one, use a check computed over the 32-symbol alphabet itself (e.g. a mod-32 scheme) so the check character stays inside the alphabet. It costs one extra character.
- The backend generates tokens with a CSPRNG (PHP `random_int`/`random_bytes`), never `mt_rand`, a HashID or an encoded id [OWASP-SM] [NIST-63B §3.2.12]. A DB unique index enforces uniqueness, with a retry on collision.

## 3. Guess-rate math

The model is the one in OWASP's worked example [OWASP-SM]: expected time to the first valid hit = `32^L / (R × N)`, where L is the token length in characters, R the attacker's guess rate and N the number of live tokens. The chance of at least one hit after G guesses is about `G × N / 32^L`.

Assumptions:

- **N = 100,000** live Invitations (the brief's "tens to hundreds of thousands") and **N = 1,000,000** (10× headroom).
- **R = 10/s** is the effective global rate an attacker gets *if* per-IP limits and bot detection on the lookup endpoint hold. For example, about 600 IPs each allowed 1 failed lookup a minute, or 10 IPs at 1/s.
- **R = 10,000/s** is OWASP's assumption, effectively "rate limiting failed or bypassed".

Expected time to the first hit (years unless stated):

| Length (chars) | Bits | N=100k, R=10/s | N=100k, R=10k/s | N=1M, R=10/s | N=1M, R=10k/s | P(hit within 1 yr), N=1M, R=10k/s |
|---|---|---|---|---|---|---|
| 8  | 40  | 0.03 | ≈18 min | 0.003 | ≈2 min | ≈1 |
| 10 | 50  | 36 | ≈13 days | 3.6 | ≈31 h | ≈1 |
| 11 | 55  | 1,140 | 1.1 | 114 | 0.11 | ≈1 |
| 12 | 60  | 36,500 | 36 | 3,650 | 3.7 | 0.27 |
| 13 | 65  | 1.2 M | 1,170 | 117,000 | 117 | 0.009 |
| 14 | 70  | 37 M | 37,000 | 3.7 M | 3,740 | 0.0003 |
| **15** | **75** | **1.2 B** | **1.2 M** | **120 M** | **120,000** | **0.000008** |
| 16 | 80  | 38 B | 38 M | 3.8 B | 3.8 M | 0.0000003 |
| 26 | 130 | ≈10^25 | ≈10^22 | ≈10^24 | ≈10^21 | ≈0 |

Reading the table:

- **11 characters or fewer is unsafe.** Safety rests entirely on rate limiting, and one bypass is enough. This is the "Gone in Six Characters" failure: researchers enumerated 5–6 character short-link tokens at scale and exposed OneDrive files (some of them writable) and shared Google Maps directions. Google lengthened its Maps tokens afterwards [GONE6].
- **12 characters (60 bits)** holds only while rate limiting holds. Without throttling, the chance of a hit within a year is about 1 in 4.
- **13 characters (65 bits)** meets the OWASP/NIST 64-bit floor. It is the shortest length that holds up even when the rate limiter fails.
- **15 characters (75 bits)** survives the worst case with about a 1000× margin, for two more characters than 13.
- "A hit" means *some* live Invitation, not a chosen one. Targeting one specific person is N times harder.

What a hit gets an attacker matters for #8. With an Invitation Token they can answer as that Employee, which skews results. If that Invitation has a **Server Draft** (Non-anonymous Questionnaire), they can also read the person's in-progress answers. That second case is the sensitive one, and it is the argument for the longer option.

## 4. Share Token

- A Share Token is meant to be passed around, so guessing one reveals little. What it does let someone do is create Invitations, and with them inject responses into a company's Open Questionnaire. The main defence is the rate limits and bot check on the intro step, already decided on the map (#1).
- There are far fewer live Share Tokens than Invitations (one per Open Questionnaire), so at the same length a Share Token is even safer.
- The map fixes one path shape `/<token>`, with the server deciding which kind it is. Using **the same length and alphabet for both** keeps the format uniform and avoids revealing the kind of link through its length.

## 5. How others do it

- **Short links (bit.ly, goo.gl, 1drv.ms; 2016):** 5–6 characters. Researchers showed they could be enumerated and that they leaked private documents and locations. The paper recommends that services detect and limit scanning, and that one leaked short URL should not expose everything else the user has shared [GONE6].
- **Qualtrics personal links:** one unique link per contact, **single use**, with a warning not to share it, and an **expiry (default 60 days)** that applies to the whole distribution [QUALTRICS]. Qualtrics does not document the length or format, so there is nothing to compare there. What carries over is the lifecycle: single use plus expiry.
- **NIST recovery codes:** short (≈20 bits), but only because of a per-account lockout after 100 failures [NIST-63B §3.2.2]. Link tokens have no such mechanism (see §1).

## 6. Leakage vectors and mitigations (beyond length)

| Vector | Source | Mitigation for this app |
|---|---|---|
| `Referer` header sent to other origins | [W3C-CAP] [REFPOL] | Send `Referrer-Policy: no-referrer` on the SPA (header from the static host and `<meta name="referrer" content="no-referrer">`), as OWASP advises for pages that carry tokens [OWASP-FP]. Browsers already default to `strict-origin-when-cross-origin` [REFPOL], which strips the path on cross-origin requests; `no-referrer` closes the remaining cases. |
| Server and CDN access logs (the static host for `form.luppa.app`; the backend API) | [W3C-CAP] | Mask the token path in the static host's access logs. Keep the token out of the API's logged request line, or mask it in the Laravel/nginx logs. Store only a keyed hash (HMAC) of the token in the DB, so a DB or log dump does not hand out live links. NIST requires hashed storage for secrets under 112 bits [NIST-63B §3.1.2.2]. |
| Error reporting (Sentry) | map #1 decision | Scrub the token from URLs, breadcrumbs and request data, as for answers. |
| Third-party scripts can read `location` | [W3C-CAP] | The bot check is the only third-party script, and it loads only on the Open Questionnaire intro (a Share Token, low value). Keep it off Invitation pages. |
| Browser history and autocomplete on **shared devices** | [W3C-CAP] | Expire tokens on submit and when the Questionnaire closes, so old history entries no longer work. Optionally call `history.replaceState()` to show a token-less URL after load; W3C notes that this costs bookmarking and resume [W3C-CAP]. Weigh it against Local Draft resume in the Draft-model ticket. |
| Mail security scanners and link previews fetching the URL | [W3C-CAP] (a GET must not trigger actions) | The first GET only reads. Never mark an Invitation "opened" or start anything on a GET. |
| Screenshots, forwarded emails, crawlers | [W3C-CAP] | `robots.txt` disallow plus `noindex`. The UI says the link is personal. Allow revoking and reissuing an Invitation's token. |
| Enumeration against the API directly (the SPA is static, so the API is the real target) | [W3C-CAP] [OWASP-FP] [GONE6] | Rate-limit **failed lookups** per IP and globally on the API endpoint. Return the same response for unknown and expired tokens. Alert on spikes of failed lookups. |

## 7. Options for #8

| Option | Example | Bits | Meets | Relies on rate limiting? |
|---|---|---|---|---|
| A: 12 chars | `7KQ3-M9XD-2HTA` | 60 | none of the floors | Yes (≈27% chance of a hit per year if bypassed, at N=1M) |
| B: 13 chars | `7KQ3M-9XD2H-TAV` | 65 | OWASP 64-bit, NIST §5.1 | Partly |
| **C: 15 chars (recommended)** | `7KQ3M-9XD2H-TAV4P` | 75 | OWASP 64-bit with ≈1000× margin | No |
| D: 26 chars | `7KQ3M-9XD2H-TAV4P-...` (26 chars) | 130 | ASVS 5.0 V7.2.3, W3C 120-bit | No, but not retypable |

Whichever option #8 picks, the following apply: CSPRNG generation, a unique index, hashed storage, normalisation (case, O/I/L, hyphens), no-referrer, scrubbing of logs and Sentry, expiry on submit and on close, revocation, and API rate limiting on failed lookups. #8 should also record why it deviates from ASVS V7.2.3 and the W3C 120-bit figure.

## Sources

- [OWASP-SM] OWASP Cheat Sheet Series, *Session Management Cheat Sheet*, "Session ID Entropy". https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- [OWASP-FP] OWASP Cheat Sheet Series, *Forgot Password Cheat Sheet*, "URL Tokens". https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- [ASVS-7] OWASP ASVS 5.0, V7 Session Management, requirement 7.2.3. https://github.com/OWASP/ASVS/blob/master/5.0/en/0x16-V7-Session-Management.md
- [NIST-63B] NIST SP 800-63B-4, *Digital Identity Guidelines: Authentication and Authenticator Management*, §3.1.2 (look-up secrets), §3.2.2 (rate limiting), §3.2.12 (random values), §5.1 (session bindings). https://pages.nist.gov/800-63-4/sp800-63b.html
- [W3C-CAP] W3C TAG, *Good Practices for Capability URLs* (draft finding). https://w3ctag.github.io/capability-urls/
- [REFPOL] W3C WebAppSec, *Referrer Policy* (editor's draft §3: default is `strict-origin-when-cross-origin`). https://w3c.github.io/webappsec-referrer-policy/
- [CROCKFORD] Douglas Crockford, *Base 32*. https://www.crockford.com/base32.html
- [GONE6] M. Georgiev, V. Shmatikov, *Gone in Six Characters: Short URLs Considered Harmful for Cloud Services*, 2016. https://arxiv.org/abs/1604.02734
- [QUALTRICS] Qualtrics Support, *Personal Links*. https://www.qualtrics.com/support/survey-platform/distributions-module/email-distribution/personal-links/
