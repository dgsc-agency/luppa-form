# luppa-form

The employee-facing form where an Employee answers a Questionnaire, taking over from the backend's Survey-form island. Vocabulary follows the backend's `CONTEXT.md` (dgsc-agency/luppa).

## Language

### Questionnaires

**Questionnaire**:
One survey run for a company: its Questions, texts, languages, type and open/closed window.
_Avoid_: Survey, form (for the run itself)

**Question**:
One item in a Questionnaire, with its input type, options and translations.

**Employee**:
The person who answers a Questionnaire. Not a login user.
_Avoid_: Respondent, user

**Invitation**:
One Employee's personal entry into one Questionnaire; every set of Answers belongs to exactly one Invitation.
_Avoid_: Link, session

**Answer**:
An Employee's response to one Question, recorded against their Invitation.

**Open Questionnaire**:
A Questionnaire reached through one shared link, where each Employee first enters their own details (org placement, age, tenure, …) and an Invitation is created for them on the spot. The backend calls this "self-created".
_Avoid_: Anonymous questionnaire, public form

**Non-anonymous Questionnaire**:
A Questionnaire whose reports may show who answered, which the Employee agrees to before answering.
_Avoid_: De-anonymized (in this repo), is_anonymous

### Links and progress

**Invitation Token**:
The short, human-retypable, case-insensitive code in a personal link that identifies one Invitation.
_Avoid_: UUID, hash, id

**Share Token**:
The short code in an Open Questionnaire's shared link that identifies the Questionnaire.

**Draft**:
The Answers an Employee has given so far, before submitting, kept so they can resume later.
_Avoid_: Progress, autosave

**Local Draft**:
A Draft kept only in the Employee's browser.

**Server Draft**:
A Draft kept by the backend; only allowed for a Non-anonymous Questionnaire.

**Theme**:
A company's look applied to the form: logo and background colour, and optionally further colours.
_Avoid_: Branding, skin
