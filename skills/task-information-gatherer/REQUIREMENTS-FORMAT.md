# Requirements file format

The output of the interview is a single markdown file capturing every business decision reached. It must be complete enough that a developer can implement the feature without making any business-logic decision on their own.

## Location

Propose `docs/requirements/<slug>.md` (kebab-case slug derived from the feature name) and confirm with the user before writing. The user may choose a different location.

## Structure

```md
# {Feature or change name}

{One or two sentences: what this change is, in plain business language.}

## Why

The problem being solved, who is asking for it, and what happens if we don't do it.

## Business value

The measurable value this adds and how we'll know it worked.

## Actors and parties

- **{Actor}** — their role, what they can do, what they trigger or are affected by.
- ...

## Business flow

The trigger, then each step as a business event, ending in the business outcome.

1. {Trigger}
2. {Step / business event}
3. ...
4. {Final outcome}

### Alternative and edge flows

- **{Scenario}** → {what the system should do}
- ...

## Business rules

- {Rule stated in business terms — eligibility, validation, timing, permissions.}
- ...

## Affected parts of the system

- {One or more of: QR Portal, Admin Portal, Client Portal, Clinician Portal, Broker Portal, Self-Onboarding form, Klarity Health Static Page, Klarity Media Articles WordPress server — or a described new part.}

## Backend work needed

**Yes** or **No**.

## Out of scope

- {What this change explicitly does not cover.}

## Open questions and assumptions

- {Anything unresolved, plus assumptions made in the absence of an answer.}
```

## Rules

- **Business language only.** No technology, data models, or implementation approach.
- **Be specific.** Replace vague terms with the canonical terms agreed during the interview.
- **Record gaps honestly.** If something wasn't resolved, list it under open questions rather than inventing an answer.
- **Omit empty sections** only when they genuinely don't apply (e.g. no alternative flows). Keep the rest even if short.
