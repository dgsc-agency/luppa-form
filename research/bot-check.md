# Bot check for the Open Questionnaire intro

Research for [#6](https://github.com/dgsc-agency/luppa-form/issues/6). The decision belongs to [#13](https://github.com/dgsc-agency/luppa-form/issues/13). Researched 2026-11-09.

## Question and context

The Open Questionnaire intro (shared link with a Share Token, where the Employee enters their own details and language, and the server creates an Invitation) needs a bot check on top of server rate limits. Which one?

Constraints from the map (#1) that bear on the choice:

- Static Vite + React SPA at `form.luppa.app`; the API is Laravel. The SPA gets the token, and Laravel verifies it on the intro `POST`.
- "No cookies", "no analytics", "no third-party scripts beyond the bot check", CORS locked to `form.luppa.app`.
- WCAG 2.2 AA; 26 UI locales.
- Employees in EU companies, answering workplace surveys. Luppa is very likely the companies' **processor**, so any bot-check vendor that receives employee data (IP address, at minimum) becomes a **sub-processor**. Under GDPR Art. 28(2) that needs the controller's prior written authorisation (specific or general, with notice of changes) ([GDPR Art. 28](https://eur-lex.europa.eu/eli/reg/2016/679/oj)).
- The threat is modest: flooding fake Invitations or junk answers that skew a company's results. Nobody gains money from it, and repeat submissions by one person are already accepted.

## Options compared

| | Cloudflare Turnstile | hCaptcha | Friendly Captcha | ALTCHA (open-source, self-hosted) | No bot check (rate limits + honeypot) |
|---|---|---|---|---|---|
| **How it works** | Client-side signals and challenges in an iframe; managed, non-interactive or invisible widget | Risk signals plus visual challenges; invisible mode on paid plans | Proof-of-work puzzle whose difficulty follows a risk score | Proof-of-work (SHA-256) challenge issued and verified by our own server | Server-side limits only |
| **Data sent to a third party** | IP, TLS fingerprint, User-Agent, sitekey/origin, to Cloudflare | IP, browser/device data, mouse, scroll, key and touch events, to hCaptcha (US) | Connection, environment and interaction info, to Friendly Captcha (DE); IP hashed | None | None |
| **Vendor role** | Processor for bot detection, **controller** for "improving" detection | Processor for End-User data; also sells labelled challenge answers | Processor | n/a | n/a |
| **Where processed** | Global; siteverify has **no EU regionalisation guarantee** | US (SCCs + DPF) | Global by default; EU-only endpoint on Advanced (€200/mo) and Enterprise | Our servers | Our servers |
| **Cookies** | None on our origin unless pre-clearance is used; Cloudflare lists CAPTCHA cookies (`cf_clearance`, `cf_chl_rc_i`, `cf_chl_rc_ni`) as strictly necessary | "Consent-exempt" cookies for anti-bot; accessibility mode needs third-party cookies | None | None | None |
| **Consent needed (ePrivacy 5(3))** | Probably not, under the CNIL's security-only reading; weakened by Cloudflare's controller-side reuse | Doubtful: the data feeds labelling and ML products, which is the reuse CNIL says breaks the exemption | Probably not | No third party involved; the exemption is easiest to argue | No |
| **Accessibility** | Vendor says WCAG 2.2 AA (the plans page says AAA); no puzzles; may show a checkbox | Vendor says WCAG 2.2 AA, but blind or impaired users must **register an email** and allow third-party cookies to skip visual puzzles | Vendor says WCAG 2.2 AA; no interaction | Vendor says WCAG 2.2 AA; no interaction; slow devices wait longer | No barrier |
| **UX friction** | Usually none; checkbox when risky; blockers, VPNs and old browsers can fail | Image puzzles on the free plan | None; a short wait on slow devices | None; a short wait on slow devices | None |
| **Laravel verification** | `POST challenges.cloudflare.com/turnstile/v0/siteverify` (`secret`, `response`, optional `remoteip`, `idempotency_key`). Token valid 300 s, single-use | `POST api.hcaptcha.com/siteverify` (form-encoded `secret`, `response`, `remoteip`, `sitekey`). Single-use, about 120 s | `POST global.frcapi.com` or `eu.frcapi.com/api/v2/captcha/siteverify` with `X-API-Key`. Single-use | `altcha-lib-php` (MIT, PHP ≥ 8.1): `createChallenge` / `verifySolution`, no outbound call. **We** keep the replay registry and add a challenge endpoint | Laravel `RateLimiter` |
| **CSP** | `script-src` and `frame-src https://challenges.cloudflare.com` (nonce / `strict-dynamic` work) | `script-src`, `frame-src`, `style-src` and `connect-src` for `https://hcaptcha.com https://*.hcaptcha.com` | `frame-src https://*.frcapi.com` (plus `connect-src` if strict) | Bundled from npm; only the worker may need `worker-src`/`blob:` under a strict policy (docs vague) | Nothing |
| **Languages** | 39 (includes hr, sr, sl, bg, ro, el, …) | Not checked | Not checked | 50+ | n/a |
| **Cost** | Free: unlimited challenges, 20 widgets, 10 hostnames/widget | Free; Pro $99–139/mo for passive mode | €9 (1k req/mo) to €200 (50k req/mo, EU endpoint); no free commercial plan | Free (MIT). Optional Cloud €47/mo (EU) or Sentinel €99/mo | Free |

## Findings

### Privacy and GDPR

- **Turnstile** collects "Signals" (client IP, TLS fingerprint, User-Agent, sitekey and origin). Cloudflare is a **processor** for bot detection but a **controller** for using the signals to improve detection, on legitimate interests ([Turnstile Privacy Addendum](https://www.cloudflare.com/turnstile-privacy-policy/)). Turnstile reportedly does not read form inputs ([Turnstile overview](https://developers.cloudflare.com/turnstile/)). In Cloudflare's Data Localization Suite, Turnstile supports the Customer Metadata Boundary but not Regional Services, and "there are no regionalization guarantees for the Siteverify API yet" ([DLS compatibility](https://developers.cloudflare.com/data-localization/compatibility/)).
- **hCaptcha** processes in the US under SCCs and the EU-US DPF. Its policy includes "To provide a market for Labeled Data", meaning challenge answers sold to customers. It collects mouse, scroll, keypress and touch events ([hCaptcha privacy policy](https://www.hcaptcha.com/privacy)).
- **Friendly Captcha** is a processor. It says it uses no HTTP cookies or persistent storage, hashes IPs one way, and stores no visitor personal data ([end-user privacy policy](https://friendlycaptcha.com/legal/privacy-end-users/)). EU-only processing (servers in Germany) requires the Advanced or Enterprise plan; the default global service "may respond from outside the EU" ([EU endpoint](https://developer.friendlycaptcha.com/docs/v2/guides/eu-endpoint)).
- **ALTCHA** open-source sends nothing to a third party. The challenge is issued and verified by our own server ([how it works](https://altcha.org/docs/how-it-works/), [server integration](https://altcha.org/docs/integration/server/)).
- **ePrivacy Art. 5(3).** The EDPB says JavaScript that "instructs the browser of the user to send asynchronous requests with the targeted information" is "gaining access" under Art. 5(3) ([Guidelines 2/2023 v2.0, para. 33](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-22023-technical-scope-art-53-eprivacy-directive_en)). So every client-side bot check, ours included, is in scope. Whether it is exempt from consent is left to national guidance (paras. 4, 40). The CNIL's cookie FAQ (Q17) says a CAPTCHA needs no consent when it only secures the site for users, but does need it when the provider reuses the data for other purposes. CNIL also notes that a consent-gated CAPTCHA defeats itself ([CNIL FAQ cookies et traceurs](https://www.cnil.fr/fr/cookies-et-autres-traceurs/regles/cookies/FAQ)). WP29 Opinion 04/2012 §3.3 exempts security cookies only for "the service that has been explicitly requested by the user" ([WP194](https://ec.europa.eu/justice/article-29/documentation/opinion-recommendation/files/2012/wp194_en.pdf)). Inference: Turnstile's controller-side "improving" purpose and hCaptcha's data market weaken the security-only exemption. Friendly Captcha and ALTCHA fit it best. No DPA or EDPB statement names Turnstile, Friendly Captcha or ALTCHA specifically (searched; none found).
- **Sub-processor.** Turnstile, hCaptcha and Friendly Captcha each add a sub-processor to Luppa's customer DPAs (GDPR Art. 28(2), (4)). ALTCHA open-source does not. If `luppa.app` already sits behind Cloudflare, Cloudflare is probably already on that list; this needs checking.

### Accessibility (WCAG 2.2 AA)

- SC 3.3.8 Accessible Authentication (Minimum) covers authentication of existing users and "does not cover creation of a username or initiation of an account" ([Understanding 3.3.8](https://www.w3.org/WAI/WCAG22/Understanding/accessible-authentication-minimum.html)). The intro is not authentication, so 3.3.8 most likely does not apply. Any interactive fallback still has to meet the general criteria (keyboard, name/role/value, status messages).
- The W3C APA note on CAPTCHA inaccessibility recommends, in order: risk analysis, honeypots, heuristics, rate limiting and proof-of-work, which has "little accessibility impact". It notes that "there is still no single, ideal solution" ([Inaccessibility of CAPTCHA, Draft Note 2021](https://www.w3.org/TR/turingtest/)).
- All four vendors claim WCAG 2.2 AA conformance; these are self-declarations. hCaptcha is the outlier in practice. Its accessible path needs an email sign-up and third-party cookies, and Safari's cross-site tracking prevention has to be turned off ([hCaptcha accessibility](https://www.hcaptcha.com/accessibility)). That is a poor fit for employees on managed work devices.
- Turnstile can fail outright, with no accessible way around it, when ad, script or fingerprinting blockers, VPNs or proxies, old browsers, or in-app WebViews are in the path ([supported browsers](https://developers.cloudflare.com/cloudflare-challenges/reference/supported-browsers/)). Corporate networks and links opened from email or Teams make these cases realistic for this audience.

### UX friction

- Turnstile managed mode is usually invisible but can show a checkbox. Its 39 languages include most Central and Eastern European ones (hr, sr, sl, bg, ro, el, …) ([supported languages](https://developers.cloudflare.com/turnstile/reference/supported-languages/)). Compare these against our 26 locales in #13.
- Proof-of-work (Friendly Captcha, ALTCHA) costs CPU time. It can run while the Employee fills in the intro, so the wait is usually hidden. Low-end phones are the worst case.
- hCaptcha's free plan shows image puzzles; passive mode costs $99–139/mo ([pricing](https://www.hcaptcha.com/pricing)).

### Laravel verification

- **Turnstile:** server `POST` to siteverify with `secret` and `response`, plus optional `remoteip` and `idempotency_key` for safe retries. Tokens are valid 300 s and single-use (`timeout-or-duplicate`). Check `hostname` and `action` in the response ([server-side validation](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)). The SPA renders explicitly (`render=explicit`, `turnstile.render/reset/remove`) ([client-side rendering](https://developers.cloudflare.com/turnstile/get-started/client-side-rendering/)). The 300 s limit means the SPA should run `execute` when the intro is submitted, not on page load.
- **hCaptcha:** form-encoded `POST`. Send `sitekey` so tokens cannot be redeemed across sites ([hCaptcha docs](https://docs.hcaptcha.com/)).
- **Friendly Captcha:** `X-API-Key` header. The docs advise accepting the submission and alerting if siteverify returns non-200, i.e. fail open ([siteverify](https://developer.friendlycaptcha.com/docs/v2/api/siteverify)).
- **ALTCHA:** two backend pieces. A `GET` challenge endpoint (rate-limited) using `Altcha::createChallenge` with `expiresAt`, and verification in the intro handler with `Altcha::verifySolution`. There is no outbound HTTP call. We must keep a **replay registry** (e.g. `Cache::add(hash, …, ttl)`); the suggested expiry is 20 min–1 h ([security recommendations](https://altcha.org/docs/integration/security-recommendations/), [altcha-lib-php](https://github.com/altcha-org/altcha-lib-php)). There is no official Laravel package.

### CSP

- Turnstile: `script-src` and `frame-src https://challenges.cloudflare.com`; nonce and `strict-dynamic` supported ([CSP reference](https://developers.cloudflare.com/turnstile/reference/content-security-policy/)).
- hCaptcha: `https://hcaptcha.com https://*.hcaptcha.com` in `script-src`, `frame-src`, `style-src` and `connect-src`, with wildcards required ([hCaptcha docs](https://docs.hcaptcha.com/)). This is the widest of the four.
- Friendly Captcha: `frame-src https://*.frcapi.com` "in most cases" ([CSP guide](https://developer.friendlycaptcha.com/docs/v2/guides/csp)).
- ALTCHA: bundled with the app (`npm i altcha`), so no third-party origin. The docs only say a strict CSP "may" need extra configuration for the bundled worker and styles ([widget docs](https://altcha.org/docs/integration/widget/)). Verify this in a prototype.

### Cost

- Turnstile: free, unlimited challenges ([plans](https://developers.cloudflare.com/turnstile/plans/)).
- hCaptcha: free with puzzles; $99–139/mo for passive mode ([pricing](https://www.hcaptcha.com/pricing)).
- Friendly Captcha: €9/mo (1k req), €39 (5k), €200 (50k, EU endpoint), Enterprise custom; no free commercial plan ([homepage, Plans & Pricing](https://friendlycaptcha.com/#pricing)). Request caps matter if a large company runs an Open Questionnaire.
- ALTCHA: open-source core free (MIT); Cloud from €47/mo (EU-hosted), Sentinel from €99/mo ([pricing](https://altcha.org/pricing/)).

## Limits of each approach

- Proof-of-work does not *detect* bots; it only makes each request cost CPU. A determined attacker pays that cost. With per-IP and per-Share-Token rate limits it does price out cheap floods, which matches the threat here.
- Turnstile and hCaptcha detect more but depend on a US vendor, put a third-party iframe in the form, and fail closed for some real users.
- Rate limits alone fail when many employees share one corporate NAT IP. Per-IP limits must be generous, so a bot behind rotating IPs gets through easily.

## Recommendation (for #13 to decide)

1. **Default: ALTCHA open-source, self-hosted in Laravel, plus rate limits and a honeypot field.** It is the only option that keeps the "no cookies, no third parties" stance whole. It adds no sub-processor to customer DPAs, sends no employee data outside our infrastructure, has no interactive step to make accessible, costs nothing, and is fully testable in our own suite. The cost is a small backend surface: a challenge endpoint, a replay registry and a difficulty setting.
2. **If stronger detection is wanted, or abuse shows up: Turnstile.** It is free, simple to verify and has a light CSP. The trade-offs to accept explicitly: Cloudflare as a sub-processor (and a controller for improving detection), no EU regionalisation of siteverify, and hard failures for some blocker or VPN users. Use invisible/managed mode, `execute` on submit, and server-side `hostname`/`action` checks.
3. **Friendly Captcha** is the vendor option if an EU-hosted managed service matters more than cost. The EU-only endpoint needs the €200/mo Advanced plan, capped at 50k requests.
4. **Drop hCaptcha.** US processing, data reused for labelling, visual puzzles on the free plan, and an accessibility path that needs registration and third-party cookies.

Whatever the choice, keep the check swappable: one `BotCheck` seam on the client (get token) and on the server (verify token), with a test double, so switching ALTCHA ↔ Turnstile is a config change.

## Open points for #13

- Is `luppa.app` already behind Cloudflare, so Cloudflare is already a sub-processor? That would change the cost of choice 2.
- Fail-open or fail-closed when the verifier (Turnstile siteverify) is down?
- Prototype check: does ALTCHA's worker run under the CSP chosen in #14 without `blob:` in `worker-src`?
- Map our 26 locales to the bot check's languages (only matters if it shows any text).
