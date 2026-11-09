# Keeping a Local Draft safe on shared devices

Research for issue #3 (map #1). Feeds the Draft-model decision in #9; nothing here is decided.

**Question.** What do current best practices say about keeping in-progress answers (a **Local Draft**) in the browser when the device may be shared (factory tablet, reception PC)?

**Context.** Static React SPA on its own origin (`form.luppa.app`), no cookies, no server of its own. For anything other than a consented Non-anonymous Questionnaire, the Draft must stay in the browser only (no Server Draft). Shared devices are a real scenario.

Sources are primary: OWASP, MDN / web.dev, the W3C Web Crypto spec, browser-vendor docs (WebKit, Chrome, Mozilla), the EDPB and the Article 29 Working Party (WP29). This is not legal advice; national regulators may read the cookie rules more strictly.

---

## 1. Storage options and what survives what

| | `sessionStorage` | `localStorage` | IndexedDB |
|---|---|---|---|
| Scope | One tab (page session) | Whole origin | Whole origin |
| Survives reload | Yes | Yes | Yes |
| Survives closing the tab | **No** | Yes | Yes |
| Survives browser restart | Only if the browser restores the session | Yes | Yes |
| Next person on the device sees it | Only if they reopen the same tab / restored session | **Yes** | **Yes** |
| API | Sync, ~5 MiB | Sync, ~5 MiB | Async, large |

- `sessionStorage`: "A page session lasts as long as the tab or the browser is open, and survives over page reloads and restores." "Closing the tab/window ends the session and clears the data in `sessionStorage`." A new tab gets a new session, except that a page opened from another (`opener`) starts with a copy. ([MDN: Window.sessionStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage)) The "restores" part matters: with session restore ("Continue where you left off", crash recovery), a closed browser can bring the Draft back.
- OWASP: "Use the object sessionStorage instead of localStorage if persistent storage is not needed." Local storage can be read by anyone with local access, so it is only appropriate "where access to the data is not assuming authentication or authorization"; IndexedDB contents can be read or changed by anyone with access to the browser profile, so don't assume they are confidential, and treat them as untrusted input on read. ([OWASP HTML5 Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html))
- web.dev recommends IndexedDB for general data; it says to avoid `localStorage` and use `sessionStorage` with care because both are synchronous and block the main thread. ([web.dev: Storage for the web](https://web.dev/articles/storage-for-the-web)) A Draft is a few KB, so the blocking cost is negligible here. The useful difference is *lifetime*, not API.
- **Eviction.** By default storage is "best effort": the browser may delete it under storage pressure, and the whole origin goes at once. ([MDN: Storage quotas and eviction criteria](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)) Safari with tracking prevention deletes all script-written storage (IndexedDB, localStorage, sessionStorage…) for an origin after 7 days of Safari use without user interaction on it. ([WebKit blog](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/)) So a Local Draft can never be relied on for more than about a week on iOS / Safari anyway.
- **Private windows.** Data is "usually deleted when the private browsing mode ends" (MDN, above). Chrome: Incognito site data is removed when the session ends, and the session ends "when you close all Incognito windows", not one tab. ([Chrome Help](https://support.google.com/chrome/answer/95464?hl=en)) Firefox (115+) supports IndexedDB in private windows, encrypted on disk with keys held only in RAM and purged at the end of the private session. ([Firefox 115 release notes](https://www.mozilla.org/en-US/firefox/115.0/releasenotes/)) So the drafts work inside a private window, then disappear. On a kiosk left running in one long private session, they **do not** disappear between people.

**Takeaway:** On a shared device, the risk is that the *next* person opens the same browser profile. `sessionStorage` stops that once the tab is closed (unless the session is restored). `localStorage` and IndexedDB never stop it until something clears them.

## 2. Does client-side encryption help when the key is in the same browser?

Mostly no.

- A Web Crypto key created with `extractable: false` cannot be exported through `exportKey()` / `wrapKey()` ([MDN: CryptoKey.extractable](https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey/extractable)). Any script running on the origin can still *use* it to decrypt, and that includes the app itself when the next person opens the same link.
- The spec itself says key material may be "persisted to disk, possibly unencrypted", and "Any application or user that has access to the device storage may be able to recover the key material" (W3C Web Cryptography API, §6.2 [Security considerations](https://www.w3.org/TR/WebCryptoAPI/)).
- The shared-device attacker isn't someone with forensic access to the disk. It is the next person, who opens the same browser and the same app, and who probably has the Invitation link in history or autocomplete. The app will decrypt for them, just as it does for the Employee.
- The only encryption that adds something is a key that is *not* in the browser: one the Employee types in (a PIN) or one the server returns per Invitation. A PIN adds friction for every Employee. A server-held key protects only against someone who lacks the link, and the link is in the browser's history.
- OWASP ASVS 5.0 treats the problem as one of **not keeping** the data, rather than encrypting it: V14.3.3 asks that browser storage (localStorage, sessionStorage, IndexedDB, cookies) holds no sensitive data, and V14.3.1 that authenticated data is cleared from client storage when the client or session ends, with `Clear-Site-Data` as a helper and client-side clearing as the fallback. ([ASVS 5.0 V14](https://github.com/OWASP/ASVS/blob/v5.0.0/5.0/en/0x23-V14-Data-Protection.md))

**Takeaway:** Encryption with a key held in the browser is security theatre against the shared-device threat. Limiting the Draft's *lifetime* and making it easy to *clear* does more.

## 3. Expiry, clearing on submit, and "shared device" / "clear my answers" patterns

- **Clear on submit.** Once the server acknowledges the submit (the contract is idempotent), delete the Draft. This is the ASVS V14.3.1 pattern: clear client storage when the task ends, in client code, without depending on the server. Also drop it from in-memory state and `history.state`, so that Back does not show answers again.
- **Expiry / TTL.** Web Storage and IndexedDB have no built-in expiry. Store a `savedAt` / `expiresAt` value with the Draft and purge expired Drafts (for every Invitation, not only the current one) whenever the app starts. Also purge when the server reports that the Questionnaire is closed or the Invitation has been submitted.
- **`Clear-Site-Data: "storage"`** clears localStorage, sessionStorage and IndexedDB for the origin and is Baseline since September 2023 ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Clear-Site-Data)). It is a *response header*, so the static host would need a dedicated URL (for example `/clear`) to serve it. It clears *every* Draft on the device, not only the current Employee's. That is useful as a kiosk "reset" link, but too blunt for the normal per-Invitation clear.
- **"Shared device" choice.** The established precedent is login "Keep me signed in / This is a public computer". WP29 discusses it explicitly: persistent login is *not* exempt from consent "because the user may not be immediately aware of the fact that closing the browser will not clear" it, and a "remember me" checkbox next to the form is an appropriate way to obtain consent (WP29 Opinion 04/2012, §3.2). The same reasoning fits a Draft: short-lived by default, persistent only when the Employee chooses it.
- **"Clear my answers."** Make it a visible action on every page, with no hidden menu. Then a person on a shared device who can't finish can leave no trace. On resume, show that the answers are restored ("We restored the answers you saved on this device on …") with a **Start over** option. Then the next person who opens a shared link isn't silently shown, or doesn't silently continue, someone else's answers.
- **Open Questionnaire caveat.** On a factory tablet, many Employees use the *same* Share Token link. Each gets their own Invitation Token after the intro, but the device then holds *that* token's Draft. If the app sends the next person straight back to it ("resume your questionnaire?"), that person gets someone else's Invitation and answers. Drafts started from a Share Token need the shortest lifetime and an explicit "Start over / I am someone else" choice on the intro page.
- **Same origin only.** OWASP warns against hosting several applications on one origin because they share localStorage (cheat sheet, above). `form.luppa.app` as a dedicated origin already meets this. XSS is the other way in ("A single Cross Site Scripting can be used to steal all the data"). The map's "no third-party scripts beyond the bot check" helps; a strict CSP would help more.

## 4. Consent under ePrivacy / GDPR

- **Art. 5(3) ePrivacy covers localStorage / IndexedDB, not only cookies.** Under EDPB Guidelines 2/2023 (v2.0, 16 Oct 2024), "storage" means placing information on the terminal equipment by instructing software on it, including "established protocols such as browser cookie storage as well as customized software" (§35–36). The rule has no minimum duration (§37) and applies to "any information … regardless of whether or not it is personal data" (§10). So the fact that the Local Draft is anonymous does not take it outside Art. 5(3). ([EDPB Guidelines 2/2023](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-22023-technical-scope-art-53-eprivacy-directive_en)) §44 says that browser processing of local storage that never leaves the device is not "gaining access", but *storing* it is still storage, and at submit the answers do leave.
- **The "strictly necessary" exemption fits a form draft.** Art. 5(3) exempts storage "strictly necessary in order for the provider of an information society service explicitly requested by the subscriber or user to provide the service". WP29 names "user-input" storage as the textbook case: it keeps "track of the user's input when filling online forms over several pages" and is "exempted under CRITERION B" (Opinion 04/2012, §3.1). ([WP29 Opinion 04/2012, WP194](https://ec.europa.eu/justice/article-29/documentation/opinion-recommendation/files/2012/wp194_en.pdf))
- **But the exemption is tied to a short lifetime.** WP29's summary: user-input storage is exempt "for the duration of a session or persistent cookies limited to a few hours in some cases" (§5). A Local Draft that lasts days across browser restarts goes beyond that, and by the persistent-login reasoning in §3.2 it would need consent (an opt-in such as "Keep my answers on this device").
- **Also:** the exemption covers only that purpose. The Draft must not double as analytics or an identifier (the map already rules out analytics). Mention the Local Draft in the privacy notice even when it is exempt; the exemption removes consent, not transparency.
- **GDPR.** A Draft for an Invitation is linked to an Employee, so it is likely personal data in the controller's hands, even though it never leaves the browser before submit. Data protection by default (Art. 25) points to the shortest lifetime that still serves the purpose. This supports session-by-default with persistence on opt-in.

## 5. Options

| Option | Shared-device safety | Resume after closing the browser | Consent position | Cost |
|---|---|---|---|---|
| **A. Memory only** | Best | No; a reload loses everything | No storage | Fails the map's "Local Draft fallback" for poor connections |
| **B. `sessionStorage` only** | Good. Gone when the tab closes; may come back through session restore | No | Clearly exempt (session-length user input) | Simple |
| **C. Persistent (localStorage / IndexedDB) by default, with TTL + clear on submit + "Clear my answers"** | Weak. The next person can see it until expiry | Yes, until expiry (≤ 7 days on Safari) | Exempt only if TTL is "a few hours"; beyond that, likely consent | Simple |
| **D. `sessionStorage` by default + opt-in "Keep my answers on this device until …" (persistent, TTL)** | Good by default. Persistence only when the Employee chooses it | Yes, for those who opt in | Default exempt; opt-in serves as consent | Moderate (one checkbox, migrate between stores) |
| **E. Any of the above + encryption with an in-browser key** | No real gain (§2) | — | — | Complexity, no benefit |

Shared by all options: clear on acknowledged submit; purge on Questionnaire closed / already submitted; key Drafts per Invitation Token; "Clear my answers" always visible; "restored answers, Start over?" prompt; validate Draft contents on read (OWASP); optional `/clear` kiosk-reset URL that serves `Clear-Site-Data: "storage"`.

## 6. Recommendation (for #9 to decide)

**Option D.**

- Default store is `sessionStorage`, keyed per Invitation Token. This covers reloads, flaky connections and accidental navigation, and closing the tab leaves nothing behind. It is clearly within the user-input exemption, so no consent prompt is needed.
- Opt-in checkbox near the progress / save area: *"Keep my answers on this device until <date>. Don't tick this on a shared device."* When ticked, the Draft moves to IndexedDB (or localStorage; small enough either way) with an `expiresAt` of a few days. Offer it only for personal Invitation links. Don't offer it for Drafts started from a Share Token (Open Questionnaire), where the device is most likely shared.
- Always: delete on acknowledged submit, purge expired / closed / submitted Drafts at startup, a visible "Clear my answers" on every page, and a "restored answers, Start over?" prompt on resume.
- No client-side encryption.
- Optional: a `/clear` URL serving `Clear-Site-Data: "storage"` that companies can bookmark on kiosk devices.

**Open points for #9.** The exact TTL for the opt-in; whether the default should instead be a few-hours persistent store (option C with a short TTL, also within WP29's wording) if losing the Draft on tab close turns out too painful in testing; and the exact wording of the privacy-notice line.
