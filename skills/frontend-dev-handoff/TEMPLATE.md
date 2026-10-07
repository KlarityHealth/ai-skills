# Handoff template

Section structure for a developer-facing frontend handoff. Include only what applies; rename, reorder or add sections to fit the feature. Fill every placeholder with facts read from code.

````md
# {Portal}: {feature, in plain words} (frontend handoff)

**For:** the frontend developer of `{frontend repo}`{, ticket if one exists}.
**Backend:** {ticket}, {issue}, {PR}. Merged to `{default branch}` as `{merge sha}`{, deployed to …}. Decisions: {where}.
**Written:** {date}, against backend `{branch}` `{sha}` and {frontend repo} `{branch}` `{sha}`.

{One paragraph: what the backend now offers.} **Nothing on the backend needs to change for this.** If you find a
gap, tell us before you work around it.

{If it builds on an earlier handoff: link it, and name the shared rules that still apply.}

---

## Contents

## 1. What this feature is
The problem in plain words, what the user can now do, and one typical use.

## 2. Decisions
| # | Decision | What it means for the screen |
Superseded decisions point to their replacement.

## 3. Words to use on screen
| Use | For | Avoid |
Taken from the domain glossary.

## 4. Navigation and redirection rules
### 4.1 Getting to the page: the route, query keys, the button that leads there.
### 4.2 Leaving the page.
### 4.3 When something is missing or gone: | Situation | Detect | Show |

## 5. Page layout
An ASCII mock-up of the page, then each region: what it shows, where the data comes from, sort and grouping
order, columns, empty-state wording. Reuse the existing pages' components.

## 6. Buttons
| Where | Button | Style | Enabled when | Does |
Hide versus disable for each conditional button. Changes to existing buttons or tooltips.

## 7. {Derived state}: how to sort each row
Include when the backend sends raw state instead of flags.
A first-match table: | # | When | Tab/section | Label | Actions offered | Side effect |
Why each ordering choice holds. Clock and recompute rules. Unknown values.

## 8. {The main action}, step by step
The confirm wording for each case, then | Result | Do | for success and every refusal, including reloads.

## 9. Endpoint reference
### 9.n {Endpoint name} (new | existing, unchanged | modified)
`VERB /path`
**Request** (an http block). **Response** (JSON covering every state the screen distinguishes), then how each
example row is classified. **Fields**: | Field | Type | Notes |, nullability, what is not sent. **Errors**, with
the real error bodies.

## 10. Gotchas
One sub-section per trap: time and dates, formats, history versus current state, what to reload, clocks.

## 11. Implementation guide, with code from this repository
### 11.1 API helpers
### 11.2 Pure functions (derived state, grouping, messages)
### 11.3 The new page (a skeleton: data flow, the main interactions)
### 11.4 Changes to existing pages
### 11.5 Tests worth writing
State that the code is not compiled, and name helpers that must be created first.

## 12. Not in scope
Each item with the decision that excludes it. Open frontend choices are listed here as open.

## 13. Checklist before you open the PR
- [ ] One checkbox per behaviour the reviewer should verify.
````
