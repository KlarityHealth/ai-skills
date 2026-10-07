---
name: frontend-dev-handoff
description: Write a developer-facing frontend handoff (markdown) for backend work that is merged: decisions, navigation, layout, buttons, endpoint reference with real request/response examples, error handling, derived-state rules, and sample code in the frontend repo's own idiom. Use when the user asks for a "frontend handoff", "FE handoff", or hand-off to the frontend developer after a backend change. For a designer-facing brief without API detail, use frontend-design-brief instead.
argument-hint: "[ticket key or issue, and the frontend repo if not obvious]"
---

# Frontend developer handoff

A **handoff** is one self-contained markdown file that lets a frontend developer build the screens for a backend change **without asking the backend team anything**. Its reader writes code: endpoints, status codes, JSON, field types, error bodies and sample code are the core content, not noise.

The finished handoff passes one test: every fact in it was **read from code**, not remembered from the plan or the conversation.

## 1. Pin the sources

Collect each of these before writing a line. Done when every item has a concrete value.

- **Backend code, merged.** The handoff describes what shipped. Find the merge commit on the default branch (`gh pr view <n> --json mergeCommit`) and confirm its tree matches what was reviewed. If the work isn't merged yet, tell the user and ask whether to wait. Examples written from a plan drift from the code.
- **Frontend repo, current.** Pull its default branch (`git pull --ff-only`; refuse if local tracked changes exist, and say so). Note the commit.
- **Decisions.** The issue body and decision log (for example the GitHub issue and its decision comments), plus decisions settled in this conversation that are not yet recorded.
- **Glossary.** `CONTEXT.md` (or the repo's domain doc) for the words the screens must use.
- **Existing handoffs** for the same feature family. The new one builds on them; link them rather than repeat their shared rules.

## 2. Read the code the handoff will describe

Done when you can name the file behind every field, status code and error body you will document.

- **Backend:** the controller methods (paths, verbs, status annotations), the response records (every field, its type, its JSON format annotations), the error handlers and the error body record, the error-code enum, and the service methods that throw. Note what is deliberately **not** in a response.
- **Frontend:** the API helper file, the utility module the feature already uses, the page the new screen hangs off, the modal/toast/loader components, and how existing pages route (the query keys they pass). Sample code must call these real helpers.

## 3. Write it

Follow [TEMPLATE.md](TEMPLATE.md) for the sections. Include what applies, drop what doesn't, add feature-specific sections. [EXAMPLE.md](EXAMPLE.md) is a complete handoff written with this skill; read it when unsure how deep a section goes.

These rules carry the lessons of earlier handoffs. Apply every one that the feature touches:

- **Decisions as screen consequences.** Each decision row says what it means for the screen, not only what was decided. Superseded decisions show their replacement.
- **Derived state.** When the backend sends raw state instead of flags, give a **first-match table** (state + time → tab or section, label, actions offered, side effects), explain each ordering choice (for example "completed beats the clock"), give the pure function that implements it, and list the edge cases to unit-test. Say what happens to an unknown value.
- **Examples cover every state.** The sample response holds at least one row per state the screen distinguishes. Then show how each example row is classified by the derived-state table.
- **Field table.** Type, format, nullability, and whether `null` is sent or the key is left out. State what is not sent, and why.
- **Error bodies as they really are.** Quote the real error body shape, including its quirks (for example a body `httpStatus` that never matches the HTTP status). For each refusal: what the UI shows and whether it reloads. Branch on the error code where the API's `reason` text was written for another audience.
- **Time.** Separate display fields (local wall-clock) from comparison fields (instants). Compare instants only; never rebuild a timestamp from date + time; format plain dates as plain dates. Note clock-change behaviour if times are local.
- **Navigation.** The exact route and query keys, how the user leaves, and one row per broken arrival: missing key, entity gone, entity belonging to someone else, generic failure.
- **Buttons table.** Where, style (reuse the page's existing components), enabled when, what it does. State hide-versus-disable for each conditional button.
- **Honest sample code.** Use the frontend repo's real helpers and patterns. Say the code is not compiled. Name any helper it assumes that does not exist yet, and how to create it.

## 4. Check it

Done when every check below passes. Fix and re-check; do not hand over with an open check.

- Every endpoint the change created or modified has a request and a response example, and every example field matches the backend record (name, type, format).
- Every status code and error code the screens can receive has a row saying what the UI does.
- Every decision that affects the screen appears, with its consequence.
- Every sample-code import resolves to a real file in the frontend repo, or is flagged as to-be-created.
- The header names the backend merge commit and the frontend commit it was written against.

## 5. Save and deliver

- Save as `<TICKET>-<portal>-<feature>-fe-handoff.md` in the root of the backend checkout, unless the user names another place. Leave it untracked unless asked to commit it.
- Summarise what it covers in a few lines, and list anything you could not confirm from code.
- Offer, and do not do without a yes, to attach it to the issue as a comment.
