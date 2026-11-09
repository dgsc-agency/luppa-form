# Blade form parity: what the Survey-form island does today, and which question types are used

Research for dgsc-agency/luppa-form#7 (map: #1). The source is the backend code in
dgsc-agency/luppa, read at the local checkout. Every path below is relative to that repo, and
`file:line` points at that checkout. Usage numbers come from read-only aggregate queries against the
backend's local database (see [Usage counts](#usage-counts)). Terms follow `GLOSSARY.md`.

## TL;DR

- **Ten input types appear in Questionnaires that ran. Six of them matter:** `radio_range`, `radio_range_reverse`, `radio_number`, `input`, `ranking` (all in nearly every deep dive and pulse), plus `single_choice`, `multiple_choice` and `placeholder` (custom and exit). `custom_dropdown_search` (already dropped) and `radio_boolean` (demo company only) are the rest.
- **`radio_boolean` has never rendered in Blade**: its partial is an empty file. The Next admin can still create it, so the new app has to support it or the admin has to stop offering it.
- **Partials with no Questions:** `test`, `checkbox`, `radio_text` and `custom_radio_number`. No Question has these input types, so the new app can leave them out.
- **Attention checks are real and they bite.** Two default `type=test` Questions (ids 110 and 111, rendered as ordinary `radio_range`) sit in 135 of the 138 deep dives that ran. Since 2025-10-01, 5,351 deep-dive Invitations ended `failed` against 15,003 `finished`, so about 26% failed. A failed Invitation still stores every Answer, flagged `test_answer_ok=false`, and the Employee sees the same thank-you screen.
- **The consent checkbox gates "Start"** whenever `Questionnaire::isAnonymous()` is true. Despite the name, that means the Questionnaire is *Non-anonymous*. The checkbox's copy says "This survey is not anonymous…". Exit Questionnaires always get it.
- **The language switch** appears on the intro screen only, and only when there is more than one language and the Questionnaire is not an Open Questionnaire. It persists `employees.language_id` on the server, then reloads the page.
- **Local progress uses `localStorage`, keyed by the page URL.** One wrinkle: the ranking order goes under a single global key. A successful submit runs `localStorage.clear()`, which wipes every Draft on that origin.
- **Several Blade quirks shouldn't be copied** (see [Bugs and quirks](#bugs-and-quirks-not-to-copy)). The server's question-set check is a no-op, the thank-you `:name` placeholders never get replaced, ranking ignores the Question's options, `radio_number` ignores its `from/to` config, and blank optional texts are stored as empty Answers.

## 1. Routes and screens

| Route | Handler | Result |
|---|---|---|
| `GET /form/{uuid}` | `Form\QuestionnaireController@index` (`routes/web/form.php:9`) | Form, finished or not-available screen |
| `GET /anonymous/{questionnaire}/intro?lang=` | `QuestionnaireController@intro` (`routes/web/form.php:12`) | Open Questionnaire intro (React) |
| `GET /testForm/{questionnaire}/{employee?}?lang=` | `TestQuestionnaireController@index` (`routes/web/form.php:14`) | Admin preview of the same view |
| `POST /api/form/{uuid}` | `QuestionnaireSubmitController@index` (`routes/api/form.php:11`) | Submit |
| `POST /api/invitations/{uuid}/change-language` | `EmployeeChangeLanguageController` (`routes/api/form.php:14`) | Persist language |
| `POST /api/formTest` | closure returning `{success:true}` (`routes/api/form.php:17`) | Dry-run submit for preview |
| `GET /api/question/custom-dropdown-search/{company}` | `CustomDropdownSearchData` (`routes/api/form.php:23`) | Employee name list (dropped) |

The decision order on `GET /form/{uuid}` is `app/Http/Controllers/Form/QuestionnaireController.php:18-61`:

1. **Unknown uuid, or an Invitation with status `draft`**: the `form.not_available` view (`:22-24`).
2. The locale is set from `employee.language_id`, falling back to `company.default_language` (`:26`, `app/Models/Employee.php:149-152`).
3. **Invitation ended** (status `finished` **or `failed`**, see `app/Enums/InvitationStatus.php:32-35`): the `form.finished` view shows the Questionnaire's `text_thank_you` and the company logo (`:28-34`). Failing the attention check is therefore invisible to the Employee.
4. **`! canBeOpened()`**: the `form.not_available` view (`:36-40`). `canBeOpened` = `isOnTime() && ! hasEnded() && questionnaire.isActive()` (`app/Models/Invitation.php:187-190`).
5. Otherwise it records `started_at` on the first open (`:42-45`) and renders `form.index`.

Screens:

- **Not available**: one heading, `questionnaire.not_available_title` ("Form not available"). It uses the hardcoded Luppa logo `/images/logo.png`, not the company's (`resources/views/form/not_available.blade.php:6,12`). The same screen covers unknown, draft, not-yet-started, expired and closed Questionnaires. Nothing tells the Employee which case they hit.
- **Finished**: the company logo plus `text_thank_you` (`resources/views/form/finished.blade.php:6,11`).
- **There is no separate "expired" screen.** An expired Exit Invitation (status `expired`) is not "ended" (`InvitationStatus::ended()` is finished/failed only), so it falls through to `canBeOpened()`, which fails, so the Employee sees *not available*.

## 2. Open windows (on time)

`Invitation::isOnTime()` (`app/Models/Invitation.php:156-163`): if the Invitation has both `start_date` and `end_date`, the window is `start_date <= now <= end_date`. Otherwise it is `Questionnaire::isOnTime()` (`app/Models/Questionnaire.php:389-418`): `started_at` is set and in the past, and `expires_at` is either null or in the future.

- **Exit-survey windows.** `CreateExitInvitation` sets `start_date = now + first_invitation_time days` and `end_date = now + invitation_duration days`, both at the configured notification hour and minute (`app/Services/Next/Actions/ExitSurvey/CreateExitInvitation.php:104-121`). The defaults are 1 and 15 days (`app/DTO/Questionnaire/NotificationConfig.php:119-121`). The link therefore does not open before the first notification time. HR can extend `end_date` (`app/Http/Controllers/Next/Questionnaire/ExitSurvey/ExtendInvitationDuration.php:20-29`). `UpdateInvitationStatus` runs every five minutes and marks active Exit Invitations past `end_date` as `expired` (`app/Console/Commands/Schedule/UpdateInvitationStatus.php:20-30`, `bootstrap/app.php:96`). All 6 active Exit Questionnaires have no `expires_at`, so only the per-Invitation window applies.
- **Other types.** `RefreshQuestionnaireStatus` runs every minute and flips `active` Questionnaires past `expires_at` to `finished` (`app/Console/Commands/Schedule/RefreshQuestionnaireStatus.php:34-37`, `bootstrap/app.php:93`).

## 3. Intro screen, consent and language switch

The intro is rendered inside the form page (`resources/views/form/components/intro_message.blade.php`). The form is hidden until "Start" is clicked.

- **Texts.** It shows `questionnaire.name` as `<h1>` (`:57`), then `text_intro`, passed through `__()` with `name` = the Employee's first name and `company` = the company name (`:58`). 211 of the 367 Questionnaires that ran use `:company` in `text_intro`; only 1 uses `:name`. 85 have an empty intro text, and Blade renders the heading and button anyway.
- **Consent checkbox.** _Corrected against backend master 66d7b3b2:_ the flag was flipped in 4b45eaeb (migration `2026_10_29_120000_flip_questionnaire_anonymity_meaning.php`, `1 - is_anonymous`; exit surveys forced to 0, column default now 1). Now `is_anonymous = 1` means anonymous, and the box is shown only when `! $questionnaire->isAnonymous()` (`resources/views/form/components/intro_message.blade.php:61-70`). The label is `questionnaire.not_anonymous_text` ("This survey is not anonymous…"). "Start" is rendered `disabled` and `resources/js/questionnaire/consent.js` enables it only while the box is checked. Consent is **not sent or stored**: it gates the button client-side only. The usage counts below come from a database copy taken before the flip; in today's meaning, 364 of the 367 Questionnaires that ran are Non-anonymous (they show the box), including every pulse, deep dive and exit, and only 3 custom ones are anonymous.
- **Start** writes `localStorage["<url>-formStart"]="started"`, hides the intro and shows the form (`resources/js/questionnaire/form.js:38-46`). On reload with that key set, the intro is skipped (`form.js:63-68`), which also means the consent box is skipped.
- **Language switch.** It is rendered only when the Questionnaire has more than one language **and** `employee_source === Table` (`intro_message.blade.php:9`). Open Questionnaires choose their language on their own intro. It lives on the intro screen only; once the form is shown there is no switcher. It lists `questionnaire.languages` with the current one preselected (`:26-49`). Choosing one POSTs `{language_id}` to the change-language endpoint, then runs `window.location.reload()` (`resources/js/questionnaire/languagePicker.js:31-43`). The server checks `canBeOpened()`, checks that the language is in `questionnaire.languages`, and **updates `employees.language_id` permanently**, which also affects later emails (`app/Http/Controllers/Form/EmployeeChangeLanguageController.php:18-29`). Local progress survives the reload because the URL key does not change. In preview, the switch navigates to `?lang=` instead (`languagePicker.js:26-29`).
- **Supported locales**: 26, from `config/locales.php:13-40`. 24 of them occur in Questionnaires that ran. The top five are `en` 266, `hr` 225, `sr` 168, `sl` 30 and `bs` 18; `kk` and `hi` never occur. Most Questionnaires have 1–3 languages, and the maximum is 14.

## 4. Pages and navigation

- **Page breaks** come from `pivot_questionnaire_question.page_break`, with order from `pivot.sort` (`app/Models/Questionnaire.php:176-181`). A break goes *after* the flagged Question. The view closes the current step `div` and opens `step-N+1` (`resources/views/form/index.blade.php:46-68`). The page count is `countPageBreaks() + 1` (`index.blade.php:44`, `Questionnaire.php:656-662`). Pages in the Questionnaires that ran: 215 have a single page, 51 have 11 pages, and the maximum is 14. One Questionnaire flags its last Question, which produces an empty trailing page.
- **Buttons** (`resources/views/form/components/next_previous_buttons.blade.php`): a single page shows Submit; page 1 shows Next; the last page shows Previous and Submit; middle pages show Previous and Next.
- **The header** shows the step counter "current/total" and a progress bar with the percentage of *required* questions answered (`index.blade.php:23-35`, `form.js:674-722`). The nav becomes sticky after 220 px of scroll, and the action buttons are sticky too (`form.js:615-671`). Enter is suppressed inside the form (`form.js:18-22`).
- **Next** validates the current page (`form.js:272-463`). Every `.js-required` wrapper on the page needs a checked input, or non-empty text for `input`. On failure it shows the global `errorNotAllQuestionsAnswered` message (empty in English), puts "This is a required field." under each missing item, and scrolls to the first one. **Previous** never validates (`form.js:466-501`). Each move scrolls to the top.
- **Submit** does not run the per-page validation. It calls `calculateProgress()` and **silently does nothing** unless that returns 100 (`form.js:546-550`).

## 5. Required rules

- `questions.required` (default 1) adds `js-required` to the wrapper (`app/Models/Question.php:233-241`). Only some partials apply it: radio_range(_reverse), radio_number, single_choice, multiple_choice, input and test. **`ranking`, `placeholder` and `custom_dropdown_search` never get it.** Ranking always has a value anyway, because the initial order is submitted.
- Optional rows in Questionnaires that ran: `input` 875 of 876 (open text is effectively always optional), `placeholder` 413 of 437 (it has no input), `radio_range` 308 of 14,363, and `single_choice` 4 of 299. Everything else is always required.
- **Server** (`QuestionnaireSubmitController::isDataValid`, `:209-249`): for each submitted entry, a required Question with a missing or empty value is rejected, except when the value is `0`. The "submitted ids == Questionnaire ids" check at `:242` compares the return values of two `sort()` calls (`true !== true`), so it **never fails**. An unknown id is also not checked. Omitted Questions are silently skipped.

## 6. Comments

- Every scored type has an optional free-text comment: radio_range(_reverse), radio_number, single_choice, multiple_choice, ranking and test. Clicking the comment icon (tooltip `questionnaire.comment_description`, "Additional comments") reveals a hidden `<textarea name="{id}__comment">`. The icon only opens it, never closes it (`resources/views/form/components/comments_trigger.blade.php`, `resources/js/questionnaire/comment.js:30-56`). `input` and `placeholder` have no comment.
- Every comment is posted (`''` when unused, `form.js:503-534`). It is stored on the Answer, and multiple_choice copies it onto every selected-option row (`QuestionnaireSubmitController.php:122-141`). **Ranking comments are discarded**, because `comment` is hardcoded `null` (`:155-158`).
- Usage, as Answers with a non-empty comment: radio_range 53,354, radio_range_reverse 7,498, radio_number 1,545, multiple_choice 1,350, single_choice 997, custom_dropdown_search 3,139 and ranking 260 (historical).

## 7. Input type inventory

Rendering goes through `@include('form.questions.' . $question->input_type)` (`index.blade.php:47`). An input type with no partial throws.

| input_type | Partial | What it renders / config | Submitted value → stored |
|---|---|---|---|
| `radio_range` | `questions/radio_range.blade.php` | 6 radios with values 0–5 under Disagree…Agree headings. Each option's `data-name` (strongly/mostly/somewhat disagree, then somewhat/mostly/strongly agree) appears below once picked (`form.js:189-197`). **`config.ina=1`** swaps in the frequency labels (`*_ina` keys: "Never" … "Very often") (`:6,22-48`). | `"0".."5"` → one Answer |
| `radio_range_reverse` | `questions/radio_range_reverse.blade.php` | Same layout, but values run 5→0 left to right. The reversal is in the values, not the labels. Also supports `ina`. | one Answer |
| `radio_number` | `questions/radio_number.blade.php` | One radio per option from the translated `answers` JSON `[{label,value,description}]`, falling back to the first available locale (`Question.php:168-184`). The first option's `description` becomes the left end label and the last option's the right. For company 82 (Nelt) with no labels, the defaults are `not_likely`/`very_likely` (`:6,21-55`). **It ignores `config.from/to`**, which only the Next admin resource reads (`app/Services/Next/Resources/Questions/RadioNumber.php:41-44`). | option `value` → one Answer |
| `single_choice` | `questions/single_choice.blade.php` | Radios from `answers` `[{label,value,sort}]`. | one Answer |
| `multiple_choice` | `questions/multiple_choice.blade.php` | Checkboxes from `answers`. No min or max. | CSV → **one Answer per value** (`SubmitController:122-141`) |
| `input` | `questions/input.blade.php` | `<textarea>` with no length limit. | text → one Answer. **A blank optional text still creates an Answer with `''`** (`isset('')` at `:61`) |
| `placeholder` | `questions/placeholder.blade.php` | A speech-bubble block of text with no input, used as section intros. | nothing |
| `ranking` | `questions/ranking.blade.php` + `resources/js/questionnaire/ranking.js` | A drag-to-reorder list (SortableJS) of the 10 `Dimension::RANKING_TYPES` dimensions (`app/Models/Dimension.php:37-48`), queried with **no `orderBy`**. **It ignores the Question's own `answers`**: custom ranking Question 1426 defines 5 options and still shows all 10. There is an info icon and an illustration. The page initializes only the first `.ranking`, but one Questionnaire has two. | CSV of dimension ids in displayed order. The server reverses it and stores one Answer per dimension, with `answer` = 1 for the last up to N for the first (`:143-170`) |
| `radio_boolean` | `questions/radio_boolean.blade.php` | **An empty file, so nothing renders.** The Next admin can create it (`app/Http/Controllers/Next/Questionnaire/UpdateQuestionnaireQuestionController.php:30,52`). The HR review reads it as yes/no (`.../ExitSurvey/HrReviewController.php:138`). | — |
| `custom_dropdown_search` | `questions/custom_dropdown_search.blade.php` | Employee-name search over a public endpoint, needs at least 3 characters, and the comment becomes required once a name is picked. **Dropped** by decision in #1. | employee id |
| `test` | `questions/test.blade.php` | radio_range clone. **No Question has this input type.** | — |
| `checkbox` | `questions/checkbox.blade.php` | Static prototype with hardcoded English text. **Unused.** | — |
| `radio_text`, `custom_radio_number` | own partials | `radio_text` has `config.choice_type` single/multiple. **No Question uses either one.** | — |
| `engaged` | none | A computed row that is never rendered. On submit, the average of (value+1) across `engagement`-dimension Questions is stored against the `type=engaged` Question (`SubmitController:65-68,96-120`). | server-side |

`Question::toQuestionResource()` (`app/Models/Question.php:346-366`) maps exactly nine types: radio_range, radio_range_reverse, radio_number, ranking, input, radio_boolean, multiple_choice, single_choice and placeholder. Anything else throws. That is the set the Next admin knows about, and it matches the set that is used, minus `custom_dropdown_search`.

## 8. Attention checks (`type=test`)

- They are two default Questions: id 110 (expected answer `4`) and id 111 (expected `2`), both `input_type = radio_range` and required, with `answers` holding `{"en": <expected value>}`. They render exactly like any other agree/disagree item. `test.blade.php` is unused.
- They appear in 135 of the 138 deep dives that ran, and in all 40 recent ones. Across all Questionnaires of any status, 177 carry both checks, 26 carry one, and one carries 16 rows. No other Questionnaire type has them.
- **Effect** (`QuestionnaireSubmitController.php:49,189-207,86`): if any test answer differs from the expected value (loose `!=`), the result is `testAnswerOk = false`. Every Answer is still written with `test_answer_ok=false`, and the Invitation becomes **`failed`** instead of `finished`. `InvitationFinished` fires either way. The Employee sees the normal thank-you. Since 2025-10-01: deep dive 15,003 finished vs 5,351 failed. Per-Questionnaire `settings.response_rate_failed_as_finished` exists in code (`Questionnaire.php:746-752`), but no local Questionnaire sets `settings`.

## 9. Submit and results

- The client POSTs `{data: [{id, value, comment}]}` to `/api/form/{uuid}` with axios (`form.js:556-563`). Multi-values are joined with commas (`form.js:513-518`).
- **Success**: the form and nav are hidden and the in-page thank-you block shows (`resources/views/form/components/thank_you_block.blade.php:8-15`). Then `localStorage.clear()` runs (`form.js:564-570`).
- **Failure**: with no response, the `form_network_error` screen. With an HTTP error, the `form_submit_error` screen, which also shows the raw status, axios code and server message to the Employee (`form.js:571-606`). The current step goes back to `stepStarted`. There is no retry button and no way back to the form short of a reload.
- **Server errors** are all `400 {success:false,message}`: unknown Invitation, not openable, or invalid data (`SubmitController:31-47,251-257`). Resubmitting a finished Invitation gets 400. There is no idempotency, so a double POST before the first commits can store Answers twice.
- **Thank-you placeholders**: `thank_you_block` and `finished` pass the replacement keys as `':name'`/`':company'`. Laravel prefixes another colon (`vendor/laravel/framework/src/Illuminate/Translation/Translator.php:334-336`), so **`:company` is never substituted in thank-you texts**. 157 of the Questionnaires that ran contain `:company` there. The intro passes the keys correctly.

## 10. Local progress (today's Local Draft)

- The key prefix is the page URL without the scheme, plus `-`. That includes the query string, so in preview each `?lang=` gets its own Draft (`form.js:9-10`).
- Stored keys: `formStart`; `step-N` = `stepStarted`/`stepFinished`; `{questionId}` = the checked radio's element **id** (e.g. `123-4`), or a CSV of values for multiple_choice, which is tagged `data-type="radio_text"` (`form.js:189-224`); `{questionId}__comment`; and the open text on keyup (`form.js:227-239`).
- **The ranking order is stored under the global key `localStorage-store`**, shared by every Invitation on the device (`ranking.js:22-43`).
- On load: if `formStart` is set, the intro is skipped. It resumes at the last `stepStarted` page (the last page is forced back to started), re-checks the stored inputs, reopens comments that have text, and recomputes progress (`form.js:56-163`). There is no expiry and no "clear my answers" prompt.
- **Nothing reaches the server before submit.** A successful submit runs `localStorage.clear()`, which deletes Drafts for *all* Invitations on that browser. That matters on shared devices.

## 11. Preview (test form)

`/testForm/{questionnaire}/{employee?}?lang=` renders the same `form.index` view with `testForm: true` and posts to the dry-run `/api/formTest` (`app/Http/Controllers/Form/TestQuestionnaireController.php:13-48`). The language comes from the query string, not the Employee. If no Employee is given, it uses the first invited one, or returns 404. The new app is not asked to replace this, but the admin preview depends on it.

## 12. Open Questionnaire intro (for context)

`/anonymous/{questionnaire}/intro` shows only for an `active` Questionnaire, redirects to `?lang=<first language>`, and renders `self_created_title`/`self_created_text` (`QuestionnaireController.php:64-91`). The React app (`resources/ts/intro/introConfigForm.tsx:54-85`) fetches `/api/next/questionnaire/{id}/intro-config`, posts `/create-invitation`, and redirects to the returned form URL. Only 4 of the Questionnaires that ran use `employee_source = SelfCreated`.

## Usage counts

These come from read-only aggregate queries (`SET SESSION TRANSACTION READ ONLY`; counts only, no names, emails or answer values) against the backend's local database `luppa_live` (from `.env`). It looks like a production copy; its newest Questionnaire was created 2026-10-01. "Ran" means status `active` or `finished` and not deleted. "Recent" means created on or after 2025-10-01.

Questionnaires that ran: deep_dive 138 (40 recent), pulse 163 (33), custom 58 (48), exit 8 (5). Currently `active`: 6 exit, 1 deep dive, 1 custom.

**Questionnaires that ran containing each input type:**

| input_type | all-time | deep_dive | pulse | custom | exit | recent (dd/pulse/custom/exit) | Answer rows |
|---|---|---|---|---|---|---|---|
| radio_range | 328 | 136 | 151 | 33 | 8 | 40/33/27/5 | 4,453,174 |
| input | 328 | 133 | 151 | 36 | 8 | 40/33/33/5 | 133,311 |
| radio_number | 325 | 136 | 151 | 30 | 8 | 40/33/26/5 | 100,473 |
| radio_range_reverse | 303 | 136 | 151 | 8 | 8 | 40/33/6/5 | 528,436 |
| ranking | 139 | 135 | 0 | 4 | 0 | 40/0/2/0 | 553,167 |
| placeholder | 48 | 4 | 0 | 36 | 8 | 4/0/33/5 | — |
| single_choice | 31 | 1 | 0 | 22 | 8 | 1/0/18/5 | 39,515 |
| multiple_choice | 15 | 0 | 0 | 7 | 8 | 0/0/3/5 | 13,508 |
| custom_dropdown_search | 4 | 0 | 0 | 4 | 0 | 0/0/2/0 | 3,774 |
| radio_boolean | 4 | 0 | 0 | 4 | 0 | 0/0/2/0 | 1,432 |
| test / checkbox / radio_text / custom_radio_number | 0 | — | — | — | — | — | — |

- `radio_boolean`: all 4 Questionnaires belong to one company whose name contains "demo" (company 95). The rows come from the demo generator (`app/Services/Demo/DemoQuestionnaires.php:190-198`, `DemoAnswers.php:158`), not from the Blade form, which can't render the type.
- `custom_dropdown_search`: companies 14, 15 and 39.
- Config in use: `radio_range`/`radio_range_reverse` with `ina=1` on 60 Questions, but only 2 Questionnaires that ran. `radio_number` has `from/to` on 53 Questions: 1–4 (34), 1–10 (12), 0–10 (5), 0–5 and 1–5 (1 each). 32 have none. Option counts in `en` are 4, 5, 10 or 11, and 4 radio_number Questions have no `en` options. `radio_text.choice_type` is never set.
- Pivot rows in Questionnaires that ran: radio_range 14,363, radio_range_reverse 1,787, input 876, placeholder 437, radio_number 356, single_choice 299, ranking 139, multiple_choice 49, custom_dropdown_search 24, radio_boolean 8. Questionnaire length is 1–131 Questions, averaging 52.

## What the new app could drop

- **Drop outright**, since no Question uses them: `test`, `checkbox`, `radio_text`, `custom_radio_number`. `custom_dropdown_search` is already dropped.
- **Decide**: `radio_boolean`. It has no real usage, has never worked in Blade, but is creatable in the Next admin. Either implement it (a yes/no with values 1 and 0, matching `HrReviewController`) or ask the backend to stop offering it.
- **Low usage, keep**: `ina` labels (2 Questionnaires), multiple rankings in one Questionnaire (1).
- **Don't carry over**: the client-only consent with no record, `localStorage.clear()`, the global ranking key, ranking that ignores its options, `radio_number` that ignores `from/to`, the no-op id check, `''` Answers for blank text, the raw server error shown to the Employee, the thank-you placeholder bug, the "Not available" screen without the company logo, and the company-82 label hack.

## Bugs and quirks (not to copy)

1. `isDataValid` compares two `sort()` return values, so the Questionnaire's question set is never checked (`QuestionnaireSubmitController.php:242`).
2. The thank-you `:name`/`:company` placeholders are never replaced (`thank_you_block.blade.php:10`, `finished.blade.php:11`).
3. Ranking renders `RANKING_TYPES` in DB order and ignores `questions.answers`. Only the first ranking on a page is wired up (`ranking.js:4-8`).
4. `radio_number` reads its options from `answers` and ignores `config.from/to`, while the admin resource exposes `from/to`.
5. `calculateProgress` counts every required `input` textarea as answered whatever its content (`form.js:698-700`). Submit then fails silently below 100% (`form.js:548-550`).
6. Consent is checked only in the browser and is skipped on resume (`form.js:63-68`).
7. `localStorage.clear()` on success wipes other Invitations' Drafts. The ranking order goes under a global key.
8. `radio_number` has `$question->company_id !== '1'`, a strict comparison against a string that is always true (`radio_number.blade.php:26`), plus a hardcoded company-82 default.
9. Ranking comments are collected and then discarded (`QuestionnaireSubmitController.php:157`).
10. There is no submit idempotency, and every error is a 400 with a free-text message.
