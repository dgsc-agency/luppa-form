# Hosting and hardening the form SPA on Laravel Forge

Research for issue #5. The decision belongs to the grilling ticket #14; this file gives the facts, a recommended header set and an nginx sketch.

**Question.** A static Vite SPA at `form.luppa.app/<token>`, on Forge-managed nginx (or behind a CDN), calls the Laravel API cross-origin with the token in the path and no cookies. What hosting and security headers are best practice?

Researched 2026-11-09 against primary sources (listed at the end). Where a claim is my inference rather than a source's statement, it says so.

---

## TL;DR

1. **CSP**: send it as an HTTP header, not a `<meta>` tag. Use an allowlist built on `default-src 'none'` with no inline script or style. A static nginx site cannot mint a real per-response nonce. Turnstile needs `script-src` and `frame-src https://challenges.cloudflare.com`. Sentry needs its ingest host in `connect-src`. Roll it out in `Report-Only` mode first, reporting to Sentry's security endpoint.
2. **Referrer-Policy: `no-referrer`.** The token is in the path. The browser default (`strict-origin-when-cross-origin`) already strips the path on cross-origin requests, so `no-referrer` is belt and braces. Check it does not break Turnstile. *Bigger leak*: the Sentry SDK puts the page URL, token included, into every event. Scrub it in `beforeSend` and `beforeBreadcrumb`.
3. **HSTS** on `form.luppa.app`: `max-age=63072000; includeSubDomains`. Leave out `preload`: preloading is decided for the whole `luppa.app` apex and is hard to undo.
4. **Framing**: `frame-ancestors 'none'` plus `X-Frame-Options: DENY` for old browsers.
5. **Permissions-Policy**: deny every powerful feature (`camera=(), microphone=(), geolocation=(), payment=(), usb=(), browsing-topics=()`, and so on).
6. **nginx**: serve `/assets/*` with `try_files $uri =404` and `Cache-Control: public, max-age=31536000, immutable`. Every other path falls back to `index.html`, sent with `Cache-Control: no-cache`. Mind nginx's `add_header` inheritance trap: put the security headers in an `include` snippet repeated in each `location`.
7. **CORS (backend), urgent and independent of this app.** Today the backend has `allowed_origins => ['*']` with `supports_credentials => true`. The backend's own comment calls that "invalid in browsers", but `fruitcake/php-cors` does not fail. It **reflects whatever `Origin` the request sent** and adds `Access-Control-Allow-Credentials: true`, so any website can make credentialed (cookie) calls to `/api/*` and read the responses. Fix: list origins explicitly. The form needs no credentials.
8. **Deploy**: a Forge site with zero-downtime releases, web directory `/dist`, and `bun install --frozen-lockfile && bun run build` in the deploy script (needs bun on the server; 10-minute limit). The alternative is building in GitHub Actions and shipping `dist/`. Each release is a fresh directory, so old hashed chunks disappear at activation. A person mid-form on a stale tab then hits 404s on lazy chunks. Handle `vite:preloadError`, or keep old assets around.
9. **CDN: not needed at launch.** The bundle is small, the audience is regional, and Turnstile and rate limiting are already in the design. Putting Cloudflare's proxy in front adds DDoS absorption and edge caching, but it terminates TLS and sees every URL, tokens included. Decide that in #14.

---

## 1. Content-Security-Policy

### What the sources say

- OWASP's CSP cheat sheet calls **strict CSP** (nonces or hashes plus `'strict-dynamic'`, with `object-src 'none'` and `base-uri 'none'`) current best practice. It warns that nonces must be unique per response and come from a real templating engine: "middleware that tags every script tag defeats the protection". It calls hashes brittle. [OWASP CSP]
- OWASP's fallback "tighter" policy: `default-src 'none'`, then explicit `script-src`, `connect-src`, `img-src` and `style-src`. [OWASP CSP]
- Deliver CSP as a header. A `<meta>` CSP cannot carry `frame-ancestors`, `sandbox` or reporting. [OWASP CSP] [MDN frame-ancestors]
- `frame-ancestors` **does not fall back to `default-src`**, so set it explicitly. [MDN frame-ancestors]
- Roll out with `Content-Security-Policy-Report-Only`. Use `report-to` with a `Reporting-Endpoints` header as the primary reporting channel, and keep the deprecated `report-uri` for older browsers. [OWASP CSP]
- Vite: `html.cspNonce` adds a nonce placeholder to the tags it emits, which the server must replace with a fresh value on every request. Vite inlines small assets as `data:` URIs, so either allow `data:` in `img-src`/`font-src` or set `build.assetsInlineLimit: 0`. **Never** allow `data:` in `script-src`. [Vite features]

### Third parties

| Service | Needs | Source |
|---|---|---|
| Cloudflare Turnstile | `script-src https://challenges.cloudflare.com`; `frame-src https://challenges.cloudflare.com`. In pre-clearance mode only, `connect-src 'self'` (we will not use pre-clearance, since it needs Cloudflare in front of our domain). Works with nonces and `'strict-dynamic'`; the nonce propagates from `api.js`. | [Turnstile CSP] |
| Sentry (errors) | `connect-src` for the DSN's ingest host: `https://o<orgId>.ingest.sentry.io`, or **`https://o<orgId>.ingest.de.sentry.io` for an EU-region org** (EU orgs cannot ingest through `sentry.io`). | [Sentry CSP reporting] [Sentry data location] |
| Sentry Session Replay | `worker-src 'self' blob:`. **Recommend not enabling Replay**: it records the DOM, meaning answers, which goes against "never log answer contents". | [Sentry Replay troubleshooting] |
| Sentry `tunnel` option | Sends events to a same-origin URL, drops `sentry_key` from the query string, avoids preflights and ad blockers. **Not available here**: the SPA has no server of its own. The tunnel could live on the API, but that is backend work. | [Sentry troubleshooting] |
| Sentry as CSP report sink | Report URL `https://o<orgId>.ingest[.de].sentry.io/api/<project>/security/?sentry_key=<key>` (optional `sentry_environment` and `sentry_release`). Use `report-uri` plus `report-to`/`Reporting-Endpoints`. | [Sentry CSP reporting] |

### Nonce vs allowlist for this app (inference)

A Vite production build emits external `<script type="module" src="/assets/…">` tags and no inline script, so an allowlist with `script-src 'self' https://challenges.cloudflare.com` and no `'unsafe-inline'` blocks injected inline script. A real nonce would need per-request HTML rewriting: nginx `sub_filter` with `$request_id`, or an edge worker. That is the "middleware tags everything" pattern OWASP warns about. With no server-rendered HTML there is nothing to protect that a nonce would add. Conclusion: an **allowlist with no inline sources** is the honest choice for a static SPA. Revisit if a CDN edge worker ever exists.

### Styles

Without `'unsafe-inline'` in `style-src`, the browser blocks `<style>` elements and `style="…"` attributes in markup. It does **not** block direct property assignment on `element.style`. [MDN style-src] React's `style={{}}` uses property assignment. Apply the theme (background colour, later more colours) through CSS custom properties set with `document.documentElement.style.setProperty(...)`, never by injecting a `<style>` tag. MDN does not explicitly cover `setProperty`, so verify it in the browser matrix (an item for the test list). Also check the component library chosen later for runtime `<style>` injection (CSS-in-JS); that would force `'unsafe-inline'`.

### Logo

`img-src` must include wherever the backend serves company logos from (S3 or CDN host, or the API host). Unknown today: it is a contract question.

## 2. Referrer-Policy and the token in the URL

- Values: `no-referrer` sends nothing. `same-origin` sends the full URL same-origin and nothing cross-origin. `strict-origin-when-cross-origin`, **the browser default**, sends the full URL same-origin and only the origin cross-origin (HTTPS to HTTPS). [MDN Referrer-Policy]
- So under the default, the path token already does not reach `api.luppa.app`, Sentry or Cloudflare through `Referer`. `no-referrer` also removes it from same-origin requests (our own `/assets/*`, which land in our own nginx logs). It costs nothing because the app does not use referrers. The header can list a fallback chain: `no-referrer, strict-origin-when-cross-origin`. [MDN Referrer-Policy]
- **Unverified**: whether Turnstile depends on `Referer`. Cloudflare's docs say nothing either way; Turnstile hostname checks are configured per sitekey and validated through `hostname` in Siteverify. [Turnstile hostname mgmt] Add a smoke test. If it breaks, `same-origin` is the next-best value.

Other places the token leaks (inference, for #14):
- **Sentry events** carry `request.url` and navigation breadcrumbs that include the page URL. Scrub the token in `beforeSend` and `beforeBreadcrumb` (replace `/<token>` with `/:token`). This leak is larger than the Referer one.
- **nginx access logs** on the SPA host and the API host record the path. Use a custom `log_format` without `$request_uri`, or `access_log off` for the SPA, or accept it under the retention policy.
- **Browser history** on shared devices. Inherent to a URL token. The shared-device design (Draft clearing) cannot remove history entries.

## 3. Strict-Transport-Security

- OWASP recommends `max-age=63072000; includeSubDomains; preload`, and warns to check preload requirements and the risk of long max-age values. [OWASP headers]
- Preload requires serving the header **on the base domain** (`luppa.app`) with `includeSubDomains` and `preload`. It applies to every subdomain, internal ones included, and "cannot easily be undone". [hstspreload.org]
- So on `form.luppa.app`, send `max-age=63072000; includeSubDomains` (which covers only names under `form.luppa.app`). Preloading `luppa.app` is a separate, org-wide decision.
- Forge issues Let's Encrypt certificates that renew automatically, which needs an active Forge subscription; a lapsed subscription plus HSTS means the site goes unreachable. [Forge domains]

## 4. Other headers

From the OWASP HTTP Headers cheat sheet [OWASP headers], applied to this app:

| Header | Value | Note |
|---|---|---|
| `X-Content-Type-Options` | `nosniff` | Needs correct `Content-Type` everywhere; nginx `mime.types` gives `.js` as `application/javascript`. |
| `X-Frame-Options` | `DENY` | Legacy companion to `frame-ancestors 'none'`. [MDN frame-ancestors] |
| `Permissions-Policy` | `accelerometer=(), autoplay=(), camera=(), display-capture=(), fullscreen=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), midi=(), payment=(), publickey-credentials-get=(), usb=(), browsing-topics=()` | `()` disables a feature for the document and all iframes; an iframe `allow` attribute cannot re-grant it. [MDN Permissions-Policy] Confirm Turnstile needs none of these: its docs list no Permissions-Policy needs. |
| `Cross-Origin-Opener-Policy` | `same-origin` | Isolates the window from cross-origin popups and openers. |
| `Cross-Origin-Resource-Policy` | `same-origin` | On our own assets. |
| `Cross-Origin-Embedder-Policy` | **don't set** | `require-corp` can break the third-party Turnstile iframe; the app gains nothing from cross-origin isolation. |
| `X-XSS-Protection` | **don't set** | Deprecated; OWASP says neither set nor disable it. |
| `Expect-CT`, `Public-Key-Pins` | **don't set** | Deprecated or unsupported. |
| `Server` | `server_tokens off;` | Hides the nginx version (OWASP: remove or genericise). |

## 5. nginx sketch

Facts this relies on:
- `add_header` directives are inherited from the outer level **"if and only if there are no `add_header` directives defined on the current level"**. A `location` that sets `Cache-Control` silently drops every server-level security header. `add_header_inherit merge` fixes this only from nginx **1.29.3**. Use an `include` snippet repeated in each location. [nginx headers]
- Without `always`, headers are added only to 2xx/3xx responses, so a 404 page would go out with no CSP. [nginx headers]
- Vite: send `Cache-Control: no-cache` on the HTML so stale asset references do not persist; hashed assets in `dist/assets/` are safe to cache forever. [Vite build]
- Forge: change the config per site (edit the nginx file, needs `site:manage-nginx`) or through a server-level Nginx Template used at site creation. An invalid template can take down nginx for every site on the server. [Forge nginx templates]

```nginx
# /etc/nginx/forge-conf/form.luppa.app/security-headers.conf  (include in every location)
add_header Content-Security-Policy "default-src 'none'; script-src 'self' https://challenges.cloudflare.com; style-src 'self'; img-src 'self' data: https://LOGO_HOST; font-src 'self'; connect-src https://api.luppa.app https://oORGID.ingest.de.sentry.io; frame-src https://challenges.cloudflare.com; manifest-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'; object-src 'none'; upgrade-insecure-requests; report-uri https://oORGID.ingest.de.sentry.io/api/PROJECT/security/?sentry_key=KEY; report-to csp" always;
add_header Reporting-Endpoints 'csp="https://oORGID.ingest.de.sentry.io/api/PROJECT/security/?sentry_key=KEY"' always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
add_header Referrer-Policy "no-referrer" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Resource-Policy "same-origin" always;
add_header Permissions-Policy "accelerometer=(), autoplay=(), camera=(), display-capture=(), fullscreen=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), midi=(), payment=(), publickey-credentials-get=(), usb=(), browsing-topics=()" always;
```

```nginx
server {
    listen 443 ssl;  http2 on;
    server_name form.luppa.app;
    root /home/forge/form.luppa.app/current/dist;   # zero-downtime: current -> releases/<n>
    index index.html;
    server_tokens off;
    charset utf-8;
    # ssl_certificate ... managed by Forge

    # access_log off;   # or a log_format without $request_uri: the path is a token

    # Hashed build output: cache forever, never fall back to index.html
    location /assets/ {
        include forge-conf/form.luppa.app/security-headers.conf;
        add_header Cache-Control "public, max-age=31536000, immutable" always;
        try_files $uri =404;
    }

    # Unhashed public/ files (favicon, robots.txt): short cache
    location ~* \.(?:ico|png|svg|txt|webmanifest)$ {
        include forge-conf/form.luppa.app/security-headers.conf;
        add_header Cache-Control "public, max-age=3600" always;
        try_files $uri =404;
    }

    # SPA fallback: /<token> and everything else -> index.html, always revalidated
    location / {
        include forge-conf/form.luppa.app/security-headers.conf;
        add_header Cache-Control "no-cache" always;
        try_files $uri /index.html;
    }

    location ~ /\.(?!well-known) { deny all; }
}
# plus Forge's port-80 server that redirects to https
```

Notes:
- `/assets/` returns a real 404 for a missing chunk. With a fallback to `index.html`, a browser asking for a stale chunk would get HTML with a 200 and fail with a MIME error.
- The `.webmanifest` regex is a placeholder; there is no PWA. Trim it to what `public/` actually contains.
- The CSP above is the target. Ship it first as `Content-Security-Policy-Report-Only`. [OWASP CSP]
- Validate with Google's CSP Evaluator, which Cloudflare recommends. [Turnstile CSP]
- Keep the CSP in this repo (say `deploy/nginx/`) next to the code that has to satisfy it, so a new third-party origin and the header change land in one PR. Inference, not sourced.

## 6. CORS on the API

What the backend has today (`dgsc-agency/luppa`, `config/cors.php`, Laravel 13 with Sanctum 4):

```php
'paths' => ['api/*', 'sanctum/csrf-cookie'],
'allowed_origins' => ['*'],
'max_age' => 86400,
'supports_credentials' => true,
```

How `fruitcake/php-cors` (behind Laravel's `HandleCors`, which runs globally [Laravel CORS]) treats that, from `CorsService::configureAllowedOrigin`:

```php
if ($this->allowAllOrigins === true && !$this->supportsCredentials) {
    $response->headers->set('Access-Control-Allow-Origin', '*');
} elseif ($this->isSingleOriginAllowed()) {
    ...
} else {
    if ($this->isCorsRequest($request) && $this->isOriginAllowed($request)) {
        $response->headers->set('Access-Control-Allow-Origin', (string) $request->headers->get('Origin'));
    }
```

With `*` and credentials, `isOriginAllowed()` is true for every origin, so the library **echoes the caller's Origin** and sends `Access-Control-Allow-Credentials: true`. Browsers accept that. The backend's TODO ("invalid in browsers") is wrong about the effect: this is not broken, it is fully permissive. If any `/api/*` route authenticates with Sanctum session cookies, any website can call it as the logged-in user and read the response; cookie `SameSite` settings are the only remaining guard. **Flag this to the backend now**, independent of this app. [php-cors source]

Recommendation for #14 and the backend tasks:
- `allowed_origins`: explicit list, `https://form.luppa.app` plus any existing first-party SPA origins (for example the admin dashboard, if it is cross-origin). A single entry makes php-cors send it statically; several make it reflect only matching origins and add `Vary: Origin`.
- `supports_credentials`: `true` only if an existing first-party cookie SPA needs it. The form never needs credentials: send `fetch(..., { credentials: 'omit' })` from the SPA. Laravel's config is global, not per path, so per-path origins need middleware or a separate config approach.
- `allowed_methods` and `allowed_headers`: narrow to what the contract uses (`GET`, `POST`, `PUT`/`PATCH`; `Content-Type`, `Accept`, `Accept-Language`, plus an idempotency-key header if the contract defines one).
- `exposed_headers`: list any header the SPA must read, for example `Retry-After` for rate limits. Non-safelisted headers are invisible to JS otherwise.
- `max_age`: 86400 is fine; Chromium caps it at 7200 and Firefox at 86400. [MDN Max-Age] JSON `POST`s always preflight, so a long max-age matters on poor connections.
- CORS is not authorization: the token is the credential. Rate limits and Turnstile verification stay server-side.

## 7. Deploy on Forge

Forge facts [Forge deployments] [Forge sites]:
- Zero-downtime deployments are **on by default for new sites and only choosable at creation**. `$CREATE_RELEASE()` clones into `releases/`, then `cd $FORGE_RELEASE_DIRECTORY`, then your steps, then `$ACTIVATE_RELEASE()` repoints the `current` symlink. Forge keeps 4 releases by default for rollback.
- The deploy script runs anything, for example `npm ci`; push-to-deploy is on by default for GitHub sites. **Deployments are capped at 10 minutes.**
- Deploy hooks (a URL with a token) or the Forge CLI (`forge deploy`, needs `FORGE_API_TOKEN` and SSH) let CI trigger deploys.
- Web directory is configurable at creation (defaults to `/public`); set it to `/dist`.
- Forge also gives a free `on-forge.com` domain proxied through Cloudflare.

Options:

| | A. Build on server (Forge deploy script) | B. Build in CI, ship `dist/` |
|---|---|---|
| Steps | `$CREATE_RELEASE()`, `cd $FORGE_RELEASE_DIRECTORY`, `bun install --frozen-lockfile`, `bun run build`, `$ACTIVATE_RELEASE()` | GitHub Actions: `bun install`, `bun test`, `bun run build`; upload `dist/` (rsync to a release dir, or commit to a deploy branch Forge pulls); trigger the hook |
| Pros | Simplest; native push-to-deploy; a failed build never activates | Tests gate the deploy; no bun or node_modules on the server; reproducible artifact |
| Cons | bun on the server; build CPU on the API's box (if shared); tests do not gate unless CI also runs and push-to-deploy is off | More moving parts; Forge's model is "clone the repo", so an artifact deploy needs a deploy branch or a custom script |

A middle path that fits Forge's model: CI runs the tests on `main`, and on success calls the Forge **deploy hook** (push-to-deploy off). The server then builds (option A). Build-time env (`VITE_API_URL`, `VITE_SENTRY_DSN`) goes through Forge's "make .env variables available to deployment script".

**Stale chunks after deploy** (inference from Forge plus Vite facts). Each release is a fresh directory, so assets from the previous build vanish when `current` flips. A person mid-form holding the old `index.html` gets 404s on lazily loaded chunks. Vite emits `vite:preloadError` for this; its documented handler reloads the page. [Vite build] For this app, a reload is only safe because the Local Draft persists answers. Options for #14: (a) handle `vite:preloadError` by saving the Draft and reloading, (b) avoid route-level code splitting (the app is small), (c) copy `releases/<previous>/dist/assets/*` into the new release before activation. (a) plus (b) seems enough.

## 8. Is a CDN worth it?

Facts: Turnstile is already Cloudflare. Forge docs note Cloudflare's universal edge certificate covers only first-level subdomains, which is fine for `form.luppa.app`. [Forge domains]

Trade-offs (inference):
- **For a CDN**: DDoS absorption and WAF rate limiting at the edge in front of the Open Questionnaire intro; global edge caching of the hashed assets; HTTP/3.
- **Against**: the bundle is small and immutable-cached after the first load; the audience is mostly employees in a few countries. The CDN terminates TLS and **sees every URL, tokens included**, so it is one more processor of personal data (a DPA concern, given "no analytics, no third-party scripts beyond the bot check"). Cloudflare can also inject its own scripts (Web Analytics, Rocket Loader, email obfuscation) unless disabled, and the CSP would block them anyway. Headers would then be set at two layers.
- **Suggestion for #14**: launch on Forge nginx alone, with the headers above. If it is ever needed, add the Cloudflare proxy (DNS-only to proxied) with features that rewrite HTML turned off, and caching limited to `/assets/*`.

---

## Open items for #14

1. Referrer-Policy `no-referrer`: smoke-test Turnstile with it.
2. Logo host for `img-src` (contract).
3. Sentry region (`ingest.sentry.io` or `ingest.de.sentry.io`), and confirm Replay stays off.
4. Build on server vs CI artifact; who owns the CSP file.
5. Stale-chunk strategy.
6. CDN now or later.
7. Backend CORS fix (urgent, separate from this app) and the narrowed CORS config.
8. Token in access logs (SPA and API) and in Sentry URLs.

## Sources

- [OWASP CSP] OWASP Content Security Policy Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html
- [OWASP headers] OWASP HTTP Security Response Headers Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html
- [MDN Referrer-Policy] https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Referrer-Policy
- [MDN Permissions-Policy] https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Permissions-Policy
- [MDN frame-ancestors] https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/frame-ancestors
- [MDN style-src] https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/style-src
- [MDN Max-Age] https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Max-Age
- [hstspreload.org] https://hstspreload.org/
- [nginx headers] ngx_http_headers_module: https://nginx.org/en/docs/http/ngx_http_headers_module.html
- [Vite build] https://vite.dev/guide/build
- [Vite features] CSP section: https://vite.dev/guide/features
- [Turnstile CSP] https://developers.cloudflare.com/turnstile/reference/content-security-policy/
- [Turnstile hostname mgmt] https://developers.cloudflare.com/turnstile/additional-configuration/hostname-management/
- [Sentry CSP reporting] https://docs.sentry.io/platforms/javascript/security-policy-reporting/
- [Sentry troubleshooting] tunnel option: https://docs.sentry.io/platforms/javascript/troubleshooting/
- [Sentry Replay troubleshooting] https://docs.sentry.io/platforms/javascript/session-replay/troubleshooting/
- [Sentry data location] https://docs.sentry.io/organization/data-storage-location/
- [Laravel CORS] https://laravel.com/docs/12.x/routing#cors
- [php-cors source] https://github.com/fruitcake/php-cors/blob/master/src/CorsService.php
- [Forge deployments] https://laravel.com/forge/docs/sites/deployments
- [Forge sites] https://laravel.com/forge/docs/sites/the-basics
- [Forge domains] https://laravel.com/forge/docs/sites/domains
- [Forge nginx templates] https://laravel.com/forge/docs/servers/nginx-templates
- Backend config: `dgsc-agency/luppa` `config/cors.php` (read 2026-11-09)
