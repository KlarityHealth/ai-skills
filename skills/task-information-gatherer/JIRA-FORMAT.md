# Jira ticket suggestion format

After the requirements file is written, suggest the Jira tickets to create. Present them as markdown content in the conversation for the user to paste into Jira manually. Project key is **`KLAR`**. Do **not** create, edit, or transition tickets, and do **not** fabricate ticket numbers (see `docs/agents/issue-tracker.md`).

## Constraints

- **Title** — one sentence, short and to the point.
- **Description** — no more than 5 sentences.

## Structure

Always one **main task** ticket describing the overall change, followed by subtasks split by frontend/backend. Which subtasks exist depends on whether the work has a frontend slice, a backend slice, or both:

### a) Frontend-only task (backend work needed: No)

```md
**Main task**
Title: {main-ticket-title}
Description: {main-ticket-description}

**Subtask — Frontend**
Title: FE: {main-ticket-title}
Description: {frontend-specific-description}
```

### b) Frontend + backend task (backend work needed: Yes, with a UI change)

```md
**Main task**
Title: {main-ticket-title}
Description: {main-ticket-description}

**Subtask — Frontend**
Title: FE: {main-ticket-title}
Description: {frontend-specific-description}

**Subtask — Backend**
Title: BE: {main-ticket-title}
Description: {backend-specific-description}
```

### c) Backend-only task (backend work needed: Yes, no UI change)

```md
**Main task**
Title: {main-ticket-title}
Description: {main-ticket-description}

**Subtask — Backend**
Title: BE: {main-ticket-title}
Description: {backend-specific-description}
```

## Notes

- If the change touches multiple parts of the system that are genuinely separate pieces of work, suggest one main task per part, each with its own FE/BE subtasks — but prefer a single main task when the work is one cohesive feature.
- Keep descriptions in business terms; the subtask descriptions may scope the frontend/backend slice of the work but must not prescribe a technical solution.
