# Summary format

The summary records the problems found and every decision made. Write it in the same plain language as the interview. A developer reads it to build the feature. The person who gave the request reads it to check that you understood.

Use short sentences. Give real-looking examples. Leave out any section that has nothing in it. Put the button-test problems first, because they cause the most damage.

```md
# {Name of the feature, design, or bug}

**Ready for developers** / **Not ready yet** / **Review paused**

{Two or three sentences: what this is and why the business needs it.}

## Review overview

- Items checked: {number}
- Items to build now: {number} (of which smaller versions: {number})
- Items moved to later, dropped, or to find out first: {number}
- Items with problems: {number}
- Most common problem: {one plain sentence}
- Items still waiting for a decision: {number}

## The need

The problem today, who has it, and what happens if we do nothing.

## Who is involved

- **{Person or group}** — what they do and what they are allowed to do.

## What should happen

### 1. {Item, as one short sentence}

**Worth building?** {Build now / Build a smaller version now / Find out first} — {reason}
- Who uses it, and how often: {for example "about 10 clinics, twice a week"}
- Proof that people want it: {who asked, or "no proof yet"}
- Simpler way we considered: {option} — {why it was or was not chosen}
- How we will know it worked: {the number to check, and when}
- {If there is a decision record for this item: "See decision: {short name}"}

**Problems found:**
- You wrote: "{exact words}". Why this matters: {business effect}. Agreed change: {the fix, or "Still open"}.

{Or, if there were none: "No problems found in this item."}

**The story:**
0. {Before the start: how Anna finds out, and what must already be true.}
1. {Step: what Anna sees and does.}
2. ...
N. {After the end: what happens next, and who else is involved.}

**When things go differently:**
- At step {number}, {unusual situation} → {what should happen}

**Not asked, because…:** {unusual situations skipped, each with a few words why. Example: "Wrong order — the screens only allow one order."}

What starts this:                   {who does what}
What happens afterwards:            {what is different, for everyone}
If two people try at the same time: {who wins, and what the other one sees}
If someone tries again:             {what happens the second time}
If this is cancelled or reversed:   {what goes back, and what does not}

**Rules:**
- {One rule per line, in business words.}
- {Mark a rule "(our suggestion, approved)" when the person approved your suggestion.}

**Decision tables:** {For each rule that depends on two or more things, the filled table agreed with the person.}

**Limits:** {For each number in a rule, what happens at, below, and above it. Example: "Up to 3 bookings. A cancelled booking frees a place."}

**Example:** {A short story with names, dates, and numbers that shows the rules working.}

### 2. {Next item}

...

## Screens (for a design)

For each screen: what it shows normally, with nothing in it, with a lot in it, when something goes wrong, and while it waits.

## The bug (for a bug report)

- **Expected:** {what should happen}
- **Actually:** {what happens}
- **Steps:** {click by click}
- **Who and since when:** {who is affected, since when}
- **People already affected:** {what we do for them after the fix}

## Decisions about what to build

One record for every cut, drop, alternative, test, or "build anyway". Copy each record exactly as the person confirmed it.

### {Short name, for example "Fixed reminder time"}

- **Choice:** {Smaller version / Later / Drop / Find out first / Simpler way instead / Built anyway}
- **Decision:** {what we will and will not build}
- **Why:** {the reason, in the person's words}
- **What we lose:** {what users will not get, and who notices}
- **Who is told:** {who will be disappointed, and who tells them}
- **Decided by:** {name} — {also agreed by, or "Waiting for {name}"}
- **Look again when:** {the sign or date that reopens it}
- **Simpler way instead:** {if any, for example "staff send reminders by hand", with who does it}
- **Cheap test first:** {if any: the test, who runs it, and by when}

**All cuts together:** {one line: the person confirmed that what remains still solves the problem, or what was put back.}

## Not included

- {What this request does not cover, so nobody builds it by mistake.}

## Still open

- {Question} — affects item {number} — {who should answer, or "Person to confirm"}
```
