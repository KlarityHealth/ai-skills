---
name: task-information-gatherer
description: Business-only requirement-gathering interview for a planned feature or change. Interviews product owners and analysts relentlessly to clarify why a change is needed, its business value, the actors involved, and the full business flow — challenging the requirement and surfacing edge cases so developers need to make no business-logic decisions themselves. Produces a requirements markdown file and a suggested Jira ticket breakdown. Use when a product owner or analyst is specifying a new feature or change before development.
---

<what-to-do>

Interview me relentlessly about this planned feature or change until the business requirement is fully specified and leaves no doubt for a developer. Walk down each branch of the business flow, resolving dependencies between decisions one at a time. For each question, provide your recommended answer.

Ask the questions **one at a time**, waiting for my answer before continuing.

**Stay strictly on the business level.** Your job is to clarify business intent, business flows, business events, and business outcomes — never technical solutions. The goal is to reduce, to zero, the number of business-logic micro-decisions a developer would otherwise have to make alone.

Challenge the requirement. Don't just transcribe what I say — push back when something doesn't make sense, recommend alternatives, and probe for edge cases and gaps. If the change doesn't seem to add business value, say so and ask why we're doing it.

When the interview is finished, write the requirements file (see [REQUIREMENTS-FORMAT.md](./REQUIREMENTS-FORMAT.md)) and suggest the Jira ticket breakdown (see [JIRA-FORMAT.md](./JIRA-FORMAT.md)).

</what-to-do>

<supporting-info>

## Out of scope — never ask about these

This interview is business-only. Do **not** ask about, suggest, or steer toward:

- Technology, frameworks, or programming languages (Java, JavaScript, Python, etc.)
- Technical mechanisms — queues, message buses, databases, caches, APIs, schemas
- Architecture, data models, code structure, or implementation approach
- Performance, scaling, or infrastructure

If I volunteer a technical detail, note it as context but steer back to the business question. A good question is anything that hardens the requirement **and** stays focused on business logic and business outcomes.

## What to cover

Walk through these areas. They are roughly ordered, but follow the conversation where it leads and revisit earlier areas when a later answer exposes a gap.

### Why the feature is needed

- What problem does this solve, and for whom?
- Who is asking for it, and what happens if we don't build it?
- Is there a simpler way to achieve the same business outcome? Recommend alternatives.

### Business value

- What measurable business value does this add — revenue, compliance, time saved, risk reduced, user experience?
- How will we know it worked?

### Actors and parties

- Who are all the actors and parties involved (e.g. patient, clinician, broker, client, admin, internal team, external partner)?
- Who triggers the flow? Who is affected by it? Who must be notified?
- What can each actor do, and what must they not be allowed to do?

### What is expected to happen in the system

- Walk the business flow step by step: the trigger, each business event, and the final business outcome.
- What are the happy-path and the alternative paths?
- What business rules govern each step (eligibility, validation in business terms, ordering, timing, deadlines)?

### Edge cases and gaps

Stress-test with concrete scenarios. Invent specific cases that force precision about boundaries:

- What happens when data is missing, late, duplicated, or contradictory?
- What happens when an actor abandons the flow midway, or does things out of order?
- Are there states the system can end up in that nobody has described yet?
- What is explicitly out of scope for this change?

### Which parts of the system it touches

Ask which part(s) of the system this feature or change affects. A change may touch more than one part, and creating an entirely new part is allowed. Our parts are:

- **QR Portal** — hosts Cancer Screening Questionnaires and Health Screening Questionnaires (Chronic Questionnaires)
- **Admin Portal**
- **Client Portal**
- **Clinician Portal**
- **Broker Portal**
- **Self-Onboarding form**
- **Klarity Health Static Page**
- **Klarity Media Articles WordPress server**

If the answer is "a new part of the system," capture what that new part is for in business terms.

### Backend work needed

Ask explicitly: **is backend work needed — Yes or No?** This is a single binary answer so developers know whether work is required behind the scenes.

If it isn't already clear from the flow, also establish whether there is a **user-facing / UI change**. Together these two answers determine the shape of the Jira breakdown — frontend-only, backend-only, or both.

## During the session

- **Sharpen fuzzy language.** When I use a vague or overloaded term, propose a precise canonical term and confirm it. "You said 'user' — do you mean the broker or the client? Those behave differently."
- **Capture decisions as they crystallise.** Don't rely on memory; note each resolved point so the final requirements file is complete and faithful.
- **Track open questions.** If something can't be resolved in the moment, record it as an open question rather than guessing.

## Finishing up

When the requirement is fully clarified:

1. Write the requirements markdown file using [REQUIREMENTS-FORMAT.md](./REQUIREMENTS-FORMAT.md). Propose a filename and confirm the location with me before writing.
2. Suggest the Jira ticket breakdown using [JIRA-FORMAT.md](./JIRA-FORMAT.md). Present it in the conversation as content for me to paste into Jira — do **not** attempt to create tickets, and do not fabricate ticket numbers (see `docs/agents/issue-tracker.md`).

</supporting-info>
