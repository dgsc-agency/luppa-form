# GDPR: Server Drafts and an Open Questionnaire's self-entered details

Research for [#4](https://github.com/dgsc-agency/luppa-form/issues/4) (map: #1). Vocabulary follows `GLOSSARY.md`.

> **Not legal advice.** These are facts collected from primary sources so that a lawyer or DPO can sign off later. Where a statement is my own reading rather than a quote, it is marked **Reading**. Researched 2026-11-09.

## Sources

| Short name | Source | Status |
|---|---|---|
| GDPR | Regulation (EU) 2016/679, [EUR-Lex](https://eur-lex.europa.eu/eli/reg/2016/679/oj) | In force. EUR-Lex blocked automated fetches from this session. Art. 4, Art. 5 and Recital 26 were checked against the [gdpr-info.eu](https://gdpr-info.eu/) mirror. The other article references are cited from the official text without re-fetching it. |
| EDPB Consent | [EDPB Guidelines 05/2020 on consent](https://www.edpb.europa.eu/system/files/documents/files/file1/edpb_guidelines_202005_consent_en.pdf), v1.1 | Final |
| WP249 | [WP29 Opinion 2/2017 on data processing at work](https://ec.europa.eu/newsroom/article29/redirection/document/45631) | Endorsed by the EDPB |
| EDPB Pseudo | [EDPB Guidelines 01/2025 on pseudonymisation](https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf), adopted 16 Jan 2025 | Version for public consultation |
| EDPB Anon | [EDPB Guidelines 02/2026 on anonymisation](https://www.edpb.europa.eu/system/files/2026-09/edpb_guidelines_202602_anonymisation_v1_en_0.pdf), v1.0, adopted 7 Jul 2026 | **Draft** for public consultation |
| SRB | CJEU C-413/23 P *EDPS v SRB*, 4 Sep 2025 ([CURIA](https://curia.europa.eu/juris/liste.jsf?num=C-413/23)) | Final judgment. I read it through EDPB Anon §2.2 and press coverage, not the full text. |
| CNIL Diversity | [CNIL recommendation on diversity-measurement surveys at work](https://cnil.fr/sites/cnil/files/2025-06/recommandation_mesure_de_la_diversite_au_travail.pdf), Délibération 2025-028, 10 Apr 2025 | Final. Not binding, and written for diversity surveys, but it is the closest DPA guidance on employee surveys that I found. |
| WP194 | [WP29 Opinion 04/2012 on cookie consent exemption](https://ec.europa.eu/justice/article-29/documentation/opinion-recommendation/files/2012/wp194_en.pdf) | Still the reference text for exemptions under ePrivacy Art. 5(3) |
| EDPB ePD | [EDPB Guidelines 2/2023 on the technical scope of Art. 5(3) ePrivacy Directive](https://www.edpb.europa.eu/system/files/2024-10/edpb_guidelines_202302_technical_scope_art_53_eprivacydirective_v2_en_0.pdf), v2 | Final |

Page numbers below are PDF page indices, which can differ by one or two from the printed numbers.

---

## 1. Starting point: in this system, "anonymous" answers are pseudonymous personal data

The backend stores every Answer against Invitation → Employee, and "anonymity" is applied only in reporting. Before looking at Drafts, it helps to fix what that means legally:

- Recital 26: personal data that has been pseudonymised but "could be attributed to a natural person by the use of additional information" is information about an identifiable person. Only anonymous information falls outside the GDPR.
- EDPB Pseudo, executive summary (p. 3): pseudonymised data "is therefore personal". The same section adds that this "also holds true if pseudonymised data and additional information are not in the hands of the same person".
- SRB (2025) softened that last point for **recipients**. Pseudonymised data can be non-personal *for a recipient* that has no means reasonably likely to re-identify it. It stays personal for the controller that holds the key. EDPB Anon ¶11–14 (pp. 7–8) builds on this: anonymity is assessed "from each relevant entity's perspective".
- **Reading:** Luppa's backend and the employer, as controller, hold the Invitation → Employee link. For them, every Answer, including the Answers to an "anonymous" Questionnaire, is personal data. Reporting thresholds limit what *report viewers* see. They do not change what is stored.

So the GDPR already applies in full to both submitted Answers and any Draft, whatever the Questionnaire type. That includes Art. 5 principles, an Art. 6 lawful basis, Art. 13 information and data-subject rights. Art. 11 ("processing which does not require identification") cannot be relied on, because the controller *can* identify.

## 2. Server Drafts for a Non-anonymous Questionnaire (Employee ticked consent)

### 2.1 Consent from employees is weak as a lawful basis

- EDPB Consent ¶21–23 (§3.1.1, p. 9) says it is "problematic for employers to process personal data of current or future employees on the basis of consent", and that employees can give free consent only "in exceptional circumstances, when it will have no adverse consequences at all whether or not they give consent". Its example of something employees cannot freely refuse is filling out "assessment forms".
- WP249 §3.1.1 (pp. 6–7) and §6.2 (p. 23): employees are "almost never in a position to freely give, refuse or revoke consent". For most processing at work, "the legal basis ... cannot and should not be the consent of the employees". Legitimate interest is the usual alternative, and it needs a proportionality test, which may form part of a DPIA.
- CNIL Diversity art. 3 (¶15–16): for employee surveys the CNIL prefers **legitimate interest**, finding consent "always difficult to rely on" in a subordinate relationship. It keeps explicit consent only for special-category data (art. 8), and it requires participation to be genuinely voluntary under any basis (art. 5).
- Recital 43 (GDPR) says consent is not a valid basis where there is a "clear imbalance" between data subject and controller.

**Reading:** the consent checkbox shown before a Non-anonymous Questionnaire may not hold up as an Art. 6(1)(a) lawful basis. It is safer to treat it as an **informed acknowledgement plus a voluntariness safeguard**, and to state another basis (usually Art. 6(1)(f), or national employment law under Art. 88). This is a decision for the employer as controller, and lawyers should confirm it. It matters for Drafts because the map ties Server Drafts to "Employee consented".

### 2.2 If consent *is* the basis, these consequences apply

- **Withdrawal must be as easy as giving consent** (Art. 7(3)). EDPB Consent ¶113–114 (§5.2, pp. 23–24): consent given with one click in a UI must be withdrawable through the same interface without "undue effort". → The form needs a visible way to withdraw that is as easy as ticking the box.
- **On withdrawal, delete.** EDPB Consent §5.2 (pp. 24–25): processing must stop, and data "should be deleted" if no other basis justifies keeping it (Art. 17(1)(b)). → On withdrawal, the Server Draft is deleted. Whether already-submitted Answers are deleted depends on the basis the controller declared up front.
- **No silent switch of basis.** EDPB Consent §5.2 and §6 (p. 25): a controller "cannot swap from consent to other lawful bases", and cannot fall back on legitimate interest retrospectively. → Decide the basis *before* launch and state it in the Art. 13 notice.
- **Granularity** (EDPB Consent §3.1.3): separate purposes need separate consent. **Reading:** keeping a Draft so the Employee can resume is plausibly part of the same purpose, which is answering this Questionnaire, so it does not need its own tick. It does need to be mentioned in the notice.

### 2.3 Minimisation, purpose limitation and retention for Server Drafts

- Art. 5(1)(b), (c) and (e): purpose limitation; data "limited to what is necessary"; kept "no longer than is necessary". Art. 25(1)–(2): data protection by design and by default. WP249 §3.2.1 and §6.4 (pp. 8, 23–24): minimise what is "shown to the employer", store for "the minimum amount of time needed with a retention period specified", and delete when no longer needed.
- CNIL Diversity art. 5 (¶21): participants "must be able to choose whether or not to answer each question individually", and no list of participants or non-participants should be kept for individual reminders.

**Reading. What this implies for the design:**
1. A Server Draft exists only so the Employee can resume. It must **not** be visible in reports or to the employer, and must not be counted as a response or a partial response. Using it as data would be a new purpose that the Employee never chose by submitting.
2. **Delete the Server Draft on submit** (it becomes the Answers), when the Employee clears it, on consent withdrawal, and **at the latest when the Questionnaire window closes**. Add a short grace period only if it is justified and written down. A Questionnaire with no end date needs a fixed draft TTL, for example N days after the last edit.
3. A Draft should not reveal *who started but didn't finish* to anyone at the employer (compare CNIL ¶21 on participant lists).
4. Keep Draft contents out of logs, error reporting and backups, or give those their own short retention (Art. 32, and the map's "never log answer contents").

## 3. Why Server Drafts for an *anonymous* Questionnaire are a problem, and whether pseudonymous storage helps

### 3.1 The problem

- An Employee answering a Questionnaire presented as anonymous expects that what they type is not tied to them. Storing typed-but-unsubmitted answers linked to their Invitation conflicts with fairness and transparency (Art. 5(1)(a), Art. 13). It also collects data the person has not chosen to give: they may abandon the form *because* they changed their mind. That conflicts with minimisation (Art. 5(1)(c)) and with voluntariness (CNIL Diversity ¶21).
- It also creates a record of **who started a survey**, which the CNIL warns against (¶21).
- **Reading:** Submitted Answers to "anonymous" Questionnaires are already stored linked to the Employee (§1), so a Server Draft does not change the legal category of the data. It adds data that has *no* submission behind it, and stores it on the server where the Local Draft alternative does not. The map's decision of Local Draft only for anonymous Questionnaires is the data-minimising choice (Art. 25(2)).

### 3.2 Does pseudonymous storage fix it?

- **No, not for Luppa or the employer as controller.** Recital 26 and EDPB Pseudo (p. 3) say pseudonymised data remains personal data. EDPB Pseudo ¶46–47 (p. 13) treats internal pseudonymisation as a *safeguard* for minimisation and confidentiality, not as an exit from the GDPR. A draft keyed by Invitation Token, or by any random ID the server can map back to an Invitation, is pseudonymous.
- **SRB's relative approach** helps only where the holder truly lacks the means to re-identify. That is not the case when the same backend holds the Invitation → Employee link.
- **Client-side encryption** of a draft, with the key only in the browser, comes closest. But EDPB Anon ¶91 (p. 28) says encryption is "intended to be reversible and differs from anonymisation". The question is whether anyone with access could decrypt with means reasonably likely to be used. Even then, the *metadata* (this Invitation has a draft, last edited at T) stays personal and still reveals who started.
- **Reading:** pseudonymisation or encryption lowers the risk but does not remove the objections in §3.1. The draft would still be personal data, still not chosen by the Employee, and still evidence of who started. Local Draft only remains the defensible default for anonymous Questionnaires.

## 4. Open Questionnaire: self-entered details

### 4.1 Self-entered demographics are personal data in small groups

- Art. 4(1): "identifiable ... directly or indirectly". Recital 26: "singling out" counts, and so do all means reasonably likely to be used.
- CNIL Diversity art. 2.1 ¶7–10 (pp. 3–4):
  - Leaving out the name is not enough. No combination of answers may allow identification, including by joining with the HR database.
  - The CNIL's example: job title + tenure + age + sex identifies "a 31-year-old female lawyer ... in the legal department for eight months".
  - Small organisations collecting sex, age, function, contract type and tenure "can allow identification by cross-referencing".
  - Even large size alone does not guarantee anonymity, so precision must be adapted to each organisation's headcount.
  - The CNIL recommends **broad bands** (it gives an age-band example, including "prefer not to answer").
- EDPB Anon ¶65 (p. 22): the No Linkage criterion can be met for opinion-survey answers "provided that they do not contain ... demographic data which might be traced back to an individual".
- **Reading:** Org placement + age band + tenure band, collected on an Open Questionnaire, is personal data wherever the combination isolates few people. That is often true of small org units. The data is linkable to the HR roster that the employer holds. So the nameless Employee record the backend creates is **personal data, not anonymous**. Add any IP address used by rate limiting or the bot check, which also links records.

### 4.2 Reporting threshold

- EDPB Anon ¶17 (p. 10) and ¶76–78, Example 17 (p. 25): aggregate data can still "relate to" individuals. In the example, a manager is told that all team members responded and no score exceeded 3, and so learns each member's view.
- CNIL Diversity art. 12 ¶47–48: results must be published in aggregated form that makes identification impossible. Statistics must not reference "a department with a small number of people". The CNIL also notes (¶12) that a 10-person department makes anonymity hard in its context.
- **No primary source found sets a numeric minimum group size.** I found no DPA that publishes one, and the EDPB draft sets none.

### 4.3 Retention and deletion of unfinished Open-Questionnaire drafts

- Art. 5(1)(e) applies, and WP249 §6.4 asks for a specified retention period.
- CNIL Diversity art. 11 ¶43–46 (p. 12): delete individual responses once they have been aggregated, "within a reasonable time". It gives **six months after the survey closes** as an example maximum. Truly anonymous statistics can be kept indefinitely.
- CNIL Diversity ¶12 (p. 5): data that indirectly identifies people, such as unique links used to ensure one answer per person, may be kept "for a very limited period" and then unlinked.
- **Reading, applied to the Open Questionnaire flow** (intro → Invitation created → continue at token):
  - An Invitation and nameless Employee created at intro, then abandoned, contains self-entered details and nothing else. Once the Questionnaire closes it serves no purpose: **delete abandoned Invitations and Employees, and their Local or Server Draft if any, at window close**, or after a short TTL if there is no close.
  - Open Questionnaires are not Non-anonymous by consent, so under the map's rule they get Local Draft only. That also avoids server-side unfinished drafts here.
  - Rate-limit and bot-check data (IP addresses) need their own short retention and must be kept apart from Answers. CNIL's internet-survey precautions ask to "partition" answers from connection logs.

## 5. Local Draft (browser storage): ePrivacy note

These facts matter because Local Draft is the default everywhere:

- Writing to `localStorage` is "storing information ... in the terminal equipment" under ePrivacy Art. 5(3). EDPB ePD (p. 12, ¶42–47) covers local storage. Reading it back without it leaving the device is outside Art. 5(3), but once it is sent to the server, Art. 5(3) applies.
- WP194 §3.1 (p. 6): first-party "user-input" storage used "to keep track of the user's input when filling online forms over several pages" is **exempt from consent**. Lifetime must be "in direct relation to the purpose". The shopping-basket example allows persistence past the session where the user would reasonably expect to recover input (§2.3, p. 5).
- **Reading:** a Local Draft is exempt from consent if it is first-party, holds form input only, has a bounded expiry (for example the Questionnaire window or N days) and serves no other purpose. Shared devices are a known scenario, so security (Art. 32) points to a visible "clear my answers", clearing on submit, and avoiding long expiry.

## 6. Risks visible in the background facts themselves (outside Drafts)

These are flagged for the legal sign-off; the issue does not ask for them to be resolved.

1. **"Anonymous" is a misnomer for the stored data (§1).** Answers to anonymous Questionnaires are personal data linked to an Employee. If Employees are told the survey is "anonymous" without qualification, that risks breaching fairness and transparency (Art. 5(1)(a), Art. 13). The notice should say something like "your answers are stored with your identity and shown only in aggregated groups of at least N". Data-subject rights (access, erasure) also apply to these Answers.
2. **A threshold of 3 is low, and a single threshold is not enough.** EDPB Anon Example 17 shows inference from aggregates, and CNIL says small departments defeat anonymity. Repeated or filtered reports allow differencing (group A minus group A-without-X), and unanimous answers leak. Suppressing complementary cells and checking for unanimity are the usual controls. No source sets a number.
3. **Company-configurable threshold, including 0.** Per-company configuration of a privacy safeguard needs a documented basis for each company. Setting 0 relies on the "de-anonymized" basis being valid, which in turn relies on employee consent that EDPB and WP29 doubt (§2.1).
4. **Consent checkbox as lawful basis (§2.1).** If that is the declared basis, withdrawal and deletion mechanics are mandatory, and the controller cannot later switch to legitimate interest (EDPB Consent §6).
5. **Art. 88 / national law.** Member States may have specific employee-data rules (WP249 §3.2.2). Some national provisions have been challenged (CJEU C-34/21, 30 Mar 2023, on a German state law mirroring §26 BDSG; not fetched in this session). Check the national law for each customer country.
6. **DPIA.** Systematic processing of employee opinions, possibly including special-category data in free text (health or wellbeing questions → Art. 9), is a likely DPIA trigger (Art. 35; WP249 §3.2.2; CNIL Diversity art. 13). WP29's DPIA guidelines (WP248) list employees as vulnerable data subjects; I did not re-fetch them here.
7. **Open Questionnaire repeat submissions + bot check.** A third-party bot-check script processes IP and device signals. That raises Art. 5(3) ePrivacy, processor or transfer questions, and is personal data linked to self-entered details unless kept separate.
8. **Roles.** The employer is presumably the controller and Luppa a processor (Art. 28 DPA). Draft retention and deletion rules must appear in the processing agreement and the employer's notice, not only in code.

## Open questions for legal sign-off

- What is the lawful basis for (a) Non-anonymous and (b) "anonymous" Questionnaires, per company or per jurisdiction? Is the checkbox consent or acknowledgement?
- What retention period applies to Server Drafts after the window closes, and to abandoned Open-Questionnaire Invitations and Employees?
- What minimum group size and suppression rules apply, and may a company lower them?
- What exact wording should the notice use where the form says "anonymous"?
