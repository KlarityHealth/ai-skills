# Admin Portal: bookings of one office, with cancel (frontend handoff)

**For:** the frontend developer of `klarity-admin-portal-v2`.
**Backend:** KLAR-10351, GitHub #2031, PR #2032. Merged to `development` as `32c0c63c0` and deployed to dev. The
decisions are in the #2005 round 21 comment (D141–D154).
**Written:** 2026-10-06, against backend `development` `32c0c63c0` and portal `development` `120e0dd` (KLAR-10236,
the in-person campaign pages).

This document tells you what the backend now offers for the bookings list, and how to build the page into the
existing in-person campaign screens. **Nothing on the backend needs to change for this page.** If you find a gap,
tell us before you work around it.

It builds on the KLAR-10236 handoff (`KLAR-10236-admin-portal-appointment-campaigns-fe-handoff.md`). The error body,
the status codes and the "times are London times" rules there apply here too.

---

## Contents

1. [What this feature is](#1-what-this-feature-is)
2. [Decisions](#2-decisions)
3. [Words to use on screen](#3-words-to-use-on-screen)
4. [Navigation and redirection rules](#4-navigation-and-redirection-rules)
5. [Page layout](#5-page-layout)
6. [Buttons](#6-buttons)
7. [Tabs and labels: how to sort each row](#7-tabs-and-labels-how-to-sort-each-row)
8. [Cancelling a booking](#8-cancelling-a-booking)
9. [Endpoint reference](#9-endpoint-reference)
10. [Gotchas](#10-gotchas)
11. [Implementation guide, with code from this repository](#11-implementation-guide-with-code-from-this-repository)
12. [Not in scope](#12-not-in-scope)
13. [Checklist before you open the PR](#13-checklist-before-you-open-the-pr)

---

## 1. What this feature is

On an in-person skin screening campaign, employees book a slot on a clinic day at one of the client's offices. Until
now, the admin could see that a slot was **booked**, but not **who** booked it. The admin could cancel a booking only
by its Booking Reference, and no screen showed that reference. In practice the admin could not cancel anything. The
Manage page even says a booked clinic day's bookings "must be cancelled first", with no way to do it.

This feature adds:

- a **Bookings** button on each office row of the Manage page;
- a new **Bookings page** for one office, listing every booking ever made there: who, when, and its status;
- a **Cancel** button on each booking that can still be cancelled. It uses the cancel endpoint that already exists.

Typical use: "the nurse is ill on Tuesday". The admin opens the office, goes to Tuesday, and cancels each booking.
Every employee gets a cancellation email (if the appointment is still in the future), and the slots and places
become free again.

---

## 2. Decisions

These are settled. The full log, with rejected options and reasons, is in the #2005 round 21 comment.

| # | Decision | What it means for the screen |
|---|---|---|
| D141 | The admin sees the bookings of **one office**, on a page of its own, opened from the office row on the Manage page. The clinic-day slot view stays `FREE` / `BOOKED` / `BLOCKED`, with no people in it. | A new page. Don't add names to `ClinicDaySlotsPanel`. |
| D142 | The admin cancels **one booking at a time**. There is no "cancel every booking on this clinic day". | One Cancel button per row. No bulk action. |
| D143 | A row shows **name, email, date of birth and sex**. It does **not** show the phone number. It also shows the Booking Reference, the date, the time, the status, and who cancelled it and when. | The response has no phone, answers or consent. Don't ask for them. |
| D144 | The endpoint returns **every** booking of the office: all statuses, past and future, with no paging and no filter. **The frontend makes the tabs.** | One call, then split the rows on the client (section 7). |
| D145 | Cancelling uses the existing admin cancel endpoint, unchanged. | `POST …/bookings/{reference}/cancel` (section 9.2). |
| D146 → D152 | A booking keeps **its own office**. A cancelled booking stays listed under the office where it happened, even after its clinic day is moved to another office or removed. | A Cancelled row can belong to a clinic day that no longer exists, or is now at another office. Group by `date`, never by a clinic-day key. |
| D147 | A booking the employee moved to another office is listed **only under its new office**. The old office shows nothing for it. | Nothing to do; just don't be surprised. |
| D148 | **No derived flags in the response.** The frontend decides the tab, the label, whether Cancel is offered, and whether an email will go out, from `status` + `startsAt` / `endsAt` (section 7). There is no `clinicDayKey`. | Use the functions in section 11.2. |
| D149 | `cancelledBy` is `PATIENT` or `ADMIN` (or `null`). `cancelledByAdmin` holds the admin's email, or `null`. | "Cancelled by employee" or "Cancelled by jo@getklarity.io". |
| D150 | Any `ADMIN` may read the list. An unknown office, or an office of another campaign, returns **404**. An office with no bookings returns **`200 []`**. | Show an empty state for `[]`; handle 404 (section 4.3). |
| D151 | The glossary gains **No-show**: an active booking whose appointment has ended without being completed. | The label "No-show" (section 3). |

---

## 3. Words to use on screen

| Use | For | Avoid |
|---|---|---|
| **Booking** | One employee's appointment in one slot. | reservation, order, registration |
| **Booking reference** | The id the employee, HR and support quote (`reference`). | booking id, order number |
| **Booked** | An active booking whose appointment hasn't ended. | Active, Confirmed |
| **Screened** | `COMPLETED`: the nurse has submitted the findings. | Completed, Done, Attended |
| **No-show** | An active booking whose appointment time has passed without a screening. | Missed, Absent, Not screened |
| **Cancelled by employee** | `CANCELLED` + `cancelledBy: "PATIENT"`. | Cancelled by patient |
| **Cancelled by {email}** | `CANCELLED` + `cancelledBy: "ADMIN"`; show `cancelledByAdmin`. | Cancelled by Klarity |
| **Clinic day** | A date the nurse visits the office. | Session (that is the API's word) |
| **Employee** | The person who booked. | Patient, user |

---

## 4. Navigation and redirection rules

### 4.1 Getting to the page

On the Manage page (`pages/CampaignManagement/Appointments/Manage.jsx`), each office row gets a **Bookings** action,
placed **first**, before Edit and Remove.

Clicking it routes to the new page, carrying the same query keys the Manage page already uses, plus the office:

```
/CampaignManagement/Appointments/Bookings?campaignKey=<campaign key>&officeKey=<office key>
                                          &clientType=<B2B|B2C>&clientId=<id>&clientName=<name>
```

- `campaignKey` and `officeKey` are required.
- `clientType`, `clientId` and `clientName` are passed through only so the header and the way back behave like the
  Manage page.

The button is always enabled, even when the office has `placesUsed: 0`, because the page also lists cancelled
history.

### 4.2 Leaving the page

- The **back** arrow in the header calls `router.back()`, the same as on the Manage page.
- There is no other way out. Cancelling a booking keeps the admin on the page.

### 4.3 When something is missing or gone

| Situation | Detect | Show |
|---|---|---|
| `campaignKey` or `officeKey` missing from the URL | `router.isReady` and a key is empty | Error card: *"No office was given. Open it from the campaign's Offices list."*, with the back arrow |
| The campaign doesn't exist (any more) | `GET …/{campaignKey}` returns 404 | Error card: *"This campaign no longer exists. Go back to the campaign list."* |
| The office doesn't exist (any more), or belongs to another campaign | the bookings call returns 404, or the office isn't in `campaign.offices` | Error card: *"This office no longer exists."* plus a button **Back to the campaign**, which does `router.replace` to the Manage page with the same `campaignKey`, `clientType`, `clientId` and `clientName` |
| Any other load error | anything else | Error card with `adminMessage(err)` and a **Try again** button, as the Manage page does |

---

## 5. Page layout

The same frame, cards, colours and table style as the Manage page (`frame`, `SectionCard`, `RowAction`, `headerCell`).
It should look like part of the same tool.

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│ ‹  London HQ: bookings                                                             │
│    Acme — October skin screening · Acme Ltd · In-person skin screening             │
│                                                                                    │
│    PLACES               EMPLOYEES             CLINIC DATES                         │
│    3 used of 120        120 (70 m / 50 f)     Thu 15 Oct 2026 – Sat 17 Oct 2026    │
│    (117 left)                                                                      │
├────────────────────────────────────────────────────────────────────────────────────┤
│  [ Upcoming (2) ]  [ Past (2) ]  [ Cancelled (2) ]           [ 🔍 Search name… ]   │
│                                                                                    │
│  Thu 15 Oct 2026                                                      2 bookings   │
│  ┌──────┬──────────────┬──────────────────┬────────────┬─────┬──────────┬────────┐ │
│  │ TIME │ NAME         │ EMAIL            │ BORN       │ SEX │ STATUS   │ ACTION │ │
│  ├──────┼──────────────┼──────────────────┼────────────┼─────┼──────────┼────────┤ │
│  │09:00–│ Jane Smith   │ jane@acme.com    │ 12 Apr 1985│  F  │ Booked   │[✕Cancel]│ │
│  │09:30 │ ref 3a5f2ad4…│                  │            │     │          │        │ │
│  │09:30–│ Tom Brown    │ tom@acme.com     │ 3 Jan 1979 │  M  │ Booked   │[✕Cancel]│ │
│  │10:00 │ ref 7c1e9b20…│                  │            │     │          │        │ │
│  └──────┴──────────────┴──────────────────┴────────────┴─────┴──────────┴────────┘ │
│                                                                                    │
│  Fri 16 Oct 2026                                                      1 booking    │
│  …                                                                                 │
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Header card**

- Title: `{office name}: bookings`. Subtitle: `{campaign name} · {clientName} · In-person skin screening`. Both come
  from `GET …/{campaignKey}` (the call the Manage page already makes); the office is
  `campaign.offices.find(o => o.key === officeKey)`.
- Facts, using the same `Fact` component: **Places** (`placesUsed` used of `workforceSize`, `placesLeft` left),
  **Employees**, and **Clinic dates** (`firstSessionDate` – `lastSessionDate`).
- No campaign actions (Edit details, Pause) on this page.

**Bookings card**

- **Three tabs**: *Upcoming*, *Past*, *Cancelled*, each with its row count. **Upcoming** is selected when the page
  opens. The rules for which row goes in which tab are in section 7.
- **Search** (optional, your call): filters the current tab by name or email, on the client. It is not a backend
  filter.
- Inside the tab, rows are **grouped by `date`**. Each group has a header with the date (`formatClinicDate`) and its
  row count.
  - **Upcoming:** groups and rows in ascending time order (soonest first). This is the order the API sends.
  - **Past** and **Cancelled:** groups newest first, rows within a group in ascending time.
- **Columns:** Time (`from`–`to`), Name (`firstName lastName`, with the reference under it, small and grey), Email,
  Born (`dateOfBirth`), Sex (`F` / `M`), Status (the label from section 7), Action.
  - On the **Cancelled** tab, the Status column shows *Cancelled by …* and, under it, the cancel time.
  - On narrow screens, keep the table in the `overflow-x-auto` wrapper, as the Manage tables do.
- **Empty states**, using the Manage page's `EmptyState`:

  | Tab | Title | Text |
  |---|---|---|
  | Upcoming | No upcoming bookings | Employees who book at this office appear here. |
  | Past | No past bookings | Screened appointments and no-shows appear here after their time has passed. |
  | Cancelled | No cancelled bookings | Bookings cancelled by an employee or an admin appear here. |

- The reference is shown in full when the admin hovers over it (`title`). Make it easy to copy, because support
  quotes it.

---

## 6. Buttons

| Where | Button | Style | Enabled when | Does |
|---|---|---|---|---|
| Manage page, office row | **Bookings** (`fa-users`) | `RowAction`, default tone, placed first | always | Routes to the Bookings page (section 4.1) |
| Manage page, clinic-day row | **Remove**, disabled tooltip | existing | (unchanged) | Change the tooltip, see below |
| Bookings page header | **‹ Back** | existing `backButton` | always | `router.back()` |
| Bookings page, tab bar | **Upcoming / Past / Cancelled** | tab buttons | always | Switch the visible rows; no API call |
| Bookings page, row | **Cancel** (`fa-times-circle`) | `RowAction`, `tone="danger"` | the row's `canCancel` is true (section 7) | Opens the confirm modal (section 8) |
| Confirm modal | **Cancel booking** / **Keep booking** | `ConfirmModal` | while not busy | Calls the cancel endpoint, or closes |
| Error card | **Try again** / **Back to the campaign** | as on the Manage page | always | Reload, or `router.replace` to the Manage page |

Rows that can't be cancelled (`COMPLETED`, `CANCELLED`) show **no** Cancel button, not a disabled one.

**Manage page: the disabled Remove tooltip on a booked clinic day.** Today it says *"It has bookings: they must be
cancelled first"*. A **screened** booking keeps its slot for good, so a clinic day with screened bookings can never be
removed. Change the tooltip to:

> *"It has bookings. Cancel the upcoming ones on the office's Bookings page. A day with screened bookings can't be removed."*

---

## 7. Tabs and labels: how to sort each row

The backend sends **no flags** (D148). Everything is decided from three fields: `status`, `startsAt` and `endsAt`
(both ISO instants), and `now`, the browser's clock.

**The rule, in this order. The first match wins:**

| # | When | Tab | Label | Cancel offered | Cancellation email |
|---|---|---|---|---|---|
| 1 | `status === "CANCELLED"` and `cancelledBy === "PATIENT"` | Cancelled | Cancelled by employee | no | (already decided) |
| 2 | `status === "CANCELLED"` and `cancelledBy === "ADMIN"` | Cancelled | Cancelled by `{cancelledByAdmin}` | no | (already decided) |
| 3 | `status === "COMPLETED"` | Past | Screened | no | — |
| 4 | `status === "ACTIVE"` and `now < startsAt` | Upcoming | Booked | **yes** | **yes**, if the cancel lands before `startsAt` |
| 5 | `status === "ACTIVE"` and `startsAt ≤ now < endsAt` | Upcoming | Booked (in progress) | **yes** | **no** |
| 6 | `status === "ACTIVE"` and `endsAt ≤ now` | Past | No-show | **yes** | **no** |

Why it works this way:

- **`COMPLETED` beats the clock** (rule 3 before 4–6). The nurse can submit findings before the slot's start time, for
  example when the employee came early. A screened booking always goes to *Past*, even if its time is still in the
  future.
- **An admin may cancel any `ACTIVE` booking at any time** (D14), including one whose time has passed (a no-show).
  This gives the office place back.
- **No email at or after the start (E8).** The backend sends the cancellation email only if the appointment hasn't
  started when the cancel happens. It decides this with **its own clock, at the moment of cancelling**. The "email"
  column above is therefore an **estimate** for the confirm wording. A cancel made at 08:59:59 for a 09:00 slot may go
  either way.
- **Recompute while the page is open.** A row moves from Upcoming to Past when its `endsAt` passes. Recompute the tabs
  on a timer (once a minute is plenty) or at least before opening the confirm modal.
- **Compare instants, not strings.** Use `Date.parse(row.startsAt)`, never the `date`/`from` text (section 10.1).
- Any `status` or `cancelledBy` value you don't recognise: put the row in *Past* with the raw status as its label, and
  offer no Cancel. This never happens today; it keeps a future value from breaking the page.

The code for this table is in section 11.2.

---

## 8. Cancelling a booking

### 8.1 The confirm modal

Use the existing `ConfirmModal` (`component/Modal`). The `message` depends on the row (section 7, rules 4–6):

**Rule 4: upcoming, email expected**
```
Cancel Jane Smith's appointment on Thu 15 Oct 2026 at 09:00?

The slot and the office place become free, and Jane gets a cancellation email.
```

**Rule 5: in progress**
```
Cancel Jane Smith's appointment on Thu 15 Oct 2026 at 09:00?

The appointment has already started, so no cancellation email is sent. The office place becomes free.
```

**Rule 6: no-show**
```
Cancel Jane Smith's appointment on Thu 15 Oct 2026 at 09:00?

The appointment time has passed, so no email is sent. The office place becomes free again.
```

`confirmText="Cancel booking"`, `cancelText="Keep booking"`, `isLoading={cancelling}`. As on the Manage page,
ignore a second click while the call runs.

### 8.2 After the call

| Result | Do |
|---|---|
| `204` | Toast *"Booking cancelled."*, close the modal, **reload the list and the campaign** (places changed). The row moves to *Cancelled*. |
| `404` | Toast *"This booking no longer exists. The list has been reloaded."*, reload. |
| `409`, reason *"The booking is already cancelled"* | Toast *"Someone else has already cancelled this booking. The list has been reloaded."*, reload. |
| `409`, `errorCode: "KLE_APT_ALREADY_SCREENED"` | Toast *"This employee has already been screened, so the booking can't be cancelled. The list has been reloaded."*, reload. |
| `409`, `errorCode: "KLE_APT_AVAILABILITY_CHANGED"`, or reason starting *"Another change won the race"* | Toast *"Something changed at the same time. The list has been reloaded. Try again."*, reload. |
| `403` | Toast *"You don't have permission to do that."* |
| anything else | Toast `adminMessage(err)`. |

**Reload after every 409.** The existing `isReloadAndRetryError()` doesn't recognise the first three codes above, so
handle them in this page.

**Don't show the API's `reason` for `KLE_APT_ALREADY_SCREENED`.** Its text (*"This email has already been screened
in this campaign"*) was written for the employee's booking form, and makes no sense to an admin. Branch on
`errorCode`, as above.

---

## 9. Endpoint reference

Base path: `/admin/portal/campaigns/appointments`. Auth: the admin token, as for every Admin Portal call (`ADMIN`).

### 9.1 List an office's bookings (new)

`GET /admin/portal/campaigns/appointments/{campaignKey}/offices/{officeKey}/bookings`

No body and no query parameters.

**Request**

```http
GET /admin/portal/campaigns/appointments/5b0c1e2a-7d0e-4a57-9f1c-2f3a9f0e8b11/offices/9a1f3c2e-0b7d-4c1e-8e5a-3f2d1c0b9a88/bookings
Authorization: Bearer <admin token>
```

**Response `200`**: a bare array, sorted by `startsAt`, then by creation order. The example below has one row of
each kind. Assume it was read on 2026-10-16 at 12:00 UK time.

```json
[
  {
    "reference": "3a5f2ad4-6b1d-4b3f-9c2e-1f8d5a7c9e14",
    "date": "2026-10-15",
    "from": "09:00",
    "to": "09:30",
    "startsAt": "2026-10-15T08:00:00Z",
    "endsAt": "2026-10-15T08:30:00Z",
    "status": "COMPLETED",
    "firstName": "Jane",
    "lastName": "Smith",
    "email": "jane.smith@acme.com",
    "dateOfBirth": "1985-04-12",
    "sex": "FEMALE",
    "cancelledAt": null,
    "cancelledBy": null,
    "cancelledByAdmin": null
  },
  {
    "reference": "7c1e9b20-4d3a-4f8e-a1b2-6e5d4c3b2a10",
    "date": "2026-10-15",
    "from": "09:30",
    "to": "10:00",
    "startsAt": "2026-10-15T08:30:00Z",
    "endsAt": "2026-10-15T09:00:00Z",
    "status": "ACTIVE",
    "firstName": "Tom",
    "lastName": "Brown",
    "email": "tom.brown@acme.com",
    "dateOfBirth": "1979-01-03",
    "sex": "MALE",
    "cancelledAt": null,
    "cancelledBy": null,
    "cancelledByAdmin": null
  },
  {
    "reference": "b2d4f6a8-1c3e-4a5b-9d7f-0e2c4a6b8d1f",
    "date": "2026-10-16",
    "from": "10:00",
    "to": "10:30",
    "startsAt": "2026-10-16T09:00:00Z",
    "endsAt": "2026-10-16T09:30:00Z",
    "status": "CANCELLED",
    "firstName": "Priya",
    "lastName": "Patel",
    "email": "priya.patel@acme.com",
    "dateOfBirth": "1991-07-22",
    "sex": "FEMALE",
    "cancelledAt": "2026-10-14T16:02:41.513208Z",
    "cancelledBy": "PATIENT",
    "cancelledByAdmin": null
  },
  {
    "reference": "e8a6c4b2-9f1d-4e3c-8b7a-5d3f1e9c7a24",
    "date": "2026-10-17",
    "from": "11:00",
    "to": "11:30",
    "startsAt": "2026-10-17T10:00:00Z",
    "endsAt": "2026-10-17T10:30:00Z",
    "status": "ACTIVE",
    "firstName": "Sam",
    "lastName": "Okafor",
    "email": "sam.okafor@acme.com",
    "dateOfBirth": "1988-11-30",
    "sex": "MALE",
    "cancelledAt": null,
    "cancelledBy": null,
    "cancelledByAdmin": null
  },
  {
    "reference": "c5e7a9b1-2d4f-4c6e-8a0b-3f5d7e9a1c36",
    "date": "2026-10-17",
    "from": "11:30",
    "to": "12:00",
    "startsAt": "2026-10-17T10:30:00Z",
    "endsAt": "2026-10-17T11:00:00Z",
    "status": "CANCELLED",
    "firstName": "Lena",
    "lastName": "Novak",
    "email": "lena.novak@acme.com",
    "dateOfBirth": "1995-02-08",
    "sex": "FEMALE",
    "cancelledAt": "2026-10-16T07:45:10.204117Z",
    "cancelledBy": "ADMIN",
    "cancelledByAdmin": "jo@getklarity.io"
  }
]
```

Read at 2026-10-16 12:00 UK time, those rows fall like this (section 7):

| Row | Rule | Tab | Label | Cancel |
|---|---|---|---|---|
| Jane Smith | 3 | Past | Screened | — |
| Tom Brown | 6 | Past | No-show | yes, no email |
| Priya Patel | 1 | Cancelled | Cancelled by employee | — |
| Sam Okafor | 4 | Upcoming | Booked | yes, email expected |
| Lena Novak | 2 | Cancelled | Cancelled by jo@getklarity.io | — |

**Fields**

| Field | Type | Notes |
|---|---|---|
| `reference` | UUID string | The Booking Reference. The id for the cancel call. |
| `date` | `YYYY-MM-DD` | The appointment date, UK time. Group by this. |
| `from`, `to` | `HH:mm` | The appointment time, UK time, for display only. |
| `startsAt`, `endsAt` | ISO instant (UTC, `Z`) | Use these for every comparison with "now". |
| `status` | `ACTIVE` \| `CANCELLED` \| `COMPLETED` | See section 7. |
| `firstName`, `lastName`, `email` | string | As the employee entered them when booking. |
| `dateOfBirth` | `YYYY-MM-DD` | A plain date, so never put it through `new Date()` (section 10.3). |
| `sex` | `MALE` \| `FEMALE` | |
| `cancelledAt` | ISO instant or `null` | `null` unless `CANCELLED`. May carry microseconds. |
| `cancelledBy` | `PATIENT` \| `ADMIN` \| `null` | `null` unless `CANCELLED`. |
| `cancelledByAdmin` | string or `null` | The admin's email when `cancelledBy` is `ADMIN`, otherwise `null`. |

Every field is always present: `null` is sent, never left out. **There is no phone, no questionnaire answers and no
consent**, by design (D143).

**Errors**

| Code | When | Body `reason` |
|---|---|---|
| `404` | unknown campaign | `Appointment campaign {key} not found` |
| `404` | unknown office, or an office of another campaign | `Office {key} not found` |
| `401` / `403` | no admin token, or not an admin | — |

An office with no bookings returns `200` with `[]`, not a 404.

### 9.2 Cancel a booking (existing, unchanged)

`POST /admin/portal/campaigns/appointments/bookings/{reference}/cancel`

No body. Note the path: it is **not** under `{campaignKey}`.

**Request**

```http
POST /admin/portal/campaigns/appointments/bookings/7c1e9b20-4d3a-4f8e-a1b2-6e5d4c3b2a10/cancel
Authorization: Bearer <admin token>
```

**Response `204 No Content`**, with no body. On success the backend:

- marks the booking `CANCELLED` with `cancelledBy: "ADMIN"` and `cancelledByAdmin` set to your admin email;
- frees the slot and the office place (`placesUsed` goes down by one);
- removes the appointment from the nurse's Clinician Portal list;
- sends the employee a cancellation email, **only** if the appointment hasn't started yet, and stops their reminders.

**Errors.** All use the standard error body. Remember that the body's `httpStatus` field always says `400`, so use
`err.response.status`.

```json
// 404: unknown reference
{ "status": "error", "reason": "Booking 7c1e9b20-… not found", "errorCode": "KLE9000", "httpStatus": 400, … }

// 409: already cancelled (someone was faster)
{ "status": "error", "reason": "The booking is already cancelled", "errorCode": "KLE9000", "httpStatus": 400, … }

// 409: already screened
{ "status": "error", "reason": "This email has already been screened in this campaign",
  "errorCode": "KLE_APT_ALREADY_SCREENED", "httpStatus": 409, "error": "Conflict", … }

// 409: availability changed under us (a race)
{ "status": "error", "reason": "The availability has just changed; please choose again",
  "errorCode": "KLE_APT_AVAILABILITY_CHANGED", "httpStatus": 409, "error": "Conflict", … }
```

For what to show for each one, see section 8.2.

### 9.3 Campaign details (existing, unchanged; used for the header)

`GET /admin/portal/campaigns/appointments/{campaignKey}`: the call the Manage page already makes
(`getAppointmentCampaignApi`). The Bookings page uses only these fields:

```json
{
  "key": "5b0c1e2a-7d0e-4a57-9f1c-2f3a9f0e8b11",
  "name": "Acme — October skin screening",
  "firstSessionDate": "2026-10-15",
  "lastSessionDate": "2026-10-17",
  "timeZone": "Europe/London",
  "offices": [
    {
      "key": "9a1f3c2e-0b7d-4c1e-8e5a-3f2d1c0b9a88",
      "name": "London HQ",
      "country": "GB",
      "workforceSize": 120,
      "maleEmployees": 70,
      "femaleEmployees": 50,
      "placesUsed": 3,
      "placesLeft": 117
    }
  ]
}
```

`placesUsed` counts **active + screened** bookings. It is **not** the number of rows in the list, which also includes
cancelled ones. Don't use it as a row count.

---

## 10. Gotchas

### 10.1 Two kinds of time: display versus comparison

`date`, `from` and `to` are **UK wall-clock time**, already converted by the backend. Show them as they are. They are
correct on both clock-change nights, even when `to` is earlier than `from` (01:30–01:00 on 25 October).

`startsAt` and `endsAt` are **instants**. Use them, and only them, to compare with now:
`Date.now() < Date.parse(row.startsAt)`. Never build a date from `date` + `from`, because the admin's browser may not
be in the UK.

### 10.2 `cancelledAt` may carry microseconds

`"2026-10-14T16:02:41.513208Z"`. `Date.parse` and `dayjs` read it fine. Show it in UK time, for example with
`dayjs(row.cancelledAt).tz("Europe/London")` if the tz plugin is loaded, or `Intl.DateTimeFormat` with
`timeZone: "Europe/London"`.

### 10.3 `dateOfBirth` is a plain date

Format it with `formatPlainDate()` from `Utils/appointmentCampaigns.js` (dayjs reads `YYYY-MM-DD` as a local date, so
it never shifts by a day). Don't use `new Date("1985-04-12")`, which is UTC midnight and shows the previous day west
of London.

### 10.4 Group by `date`, never by clinic day

A cancelled booking can belong to a clinic day that has since been removed, or moved to another office or date
(D152). The list keeps it under the office and date where it happened. A campaign has at most one clinic day per
date, so grouping by `date` is always right. The response has no clinic-day key, on purpose.

### 10.5 The list is the whole history

It includes every cancellation, and the same employee can appear twice: cancelled, then booked again. One office holds
at most a few hundred rows. There is no paging; render them all.

### 10.6 Reload both calls after a cancel

The list (the row moves to *Cancelled*) **and** the campaign (`placesUsed` / `placesLeft` change).

### 10.7 The browser clock

Tabs and labels use the admin's clock. If it is wrong, a row can sit in the wrong tab for a while. That is cosmetic:
the backend checks everything itself when cancelling.

---

## 11. Implementation guide, with code from this repository

All of this follows the patterns of `pages/CampaignManagement/Appointments/Manage.jsx` and
`Utils/appointmentCampaigns.js` on portal `development` (`120e0dd`). Adapt freely; the behaviour in sections 4–8 is
what matters.

### 11.1 API: `API/AllApi.js`

Add next to the other `appointmentCampaignsPath` functions:

```js
/** Every booking of one office: all statuses, past and future. A bare array, sorted by startsAt. */
export const listOfficeBookingsApi = async (campaignKey, officeKey) => {
  return await GET_METHOD(`${appointmentCampaignsPath}/${campaignKey}/offices/${officeKey}/bookings`);
};

/** Cancels one booking by its reference. 204, no body. Note: not under the campaign path. */
export const cancelAppointmentBookingApi = async (reference) => {
  return await POST_METHOD(`${appointmentCampaignsPath}/bookings/${reference}/cancel`);
};
```

### 11.2 Tab and label rules: `Utils/appointmentCampaigns.js`

Pure functions, so they're easy to unit-test. They implement section 7 exactly.

```js
// --- Office bookings ---

export const BOOKING_TABS = ["UPCOMING", "PAST", "CANCELLED"];

export const BOOKING_TAB_LABELS = { UPCOMING: "Upcoming", PAST: "Past", CANCELLED: "Cancelled" };

/**
 * Where one booking row belongs, and what may be done with it. The API sends no flags: everything
 * comes from status and the two instants. COMPLETED wins over the clock, because a nurse can
 * submit findings before the slot starts.
 */
export const classifyBooking = (row, now = Date.now()) => {
  if (row.status === "CANCELLED") {
    const label =
      row.cancelledBy === "ADMIN"
        ? `Cancelled by ${row.cancelledByAdmin ?? "an admin"}`
        : "Cancelled by employee";
    return { tab: "CANCELLED", label, canCancel: false, emailOnCancel: false };
  }
  if (row.status === "COMPLETED") {
    return { tab: "PAST", label: "Screened", canCancel: false, emailOnCancel: false };
  }
  if (row.status === "ACTIVE") {
    const startsAt = Date.parse(row.startsAt);
    const endsAt = Date.parse(row.endsAt);
    if (now < startsAt) return { tab: "UPCOMING", label: "Booked", canCancel: true, emailOnCancel: true };
    if (now < endsAt) {
      return { tab: "UPCOMING", label: "Booked (in progress)", canCancel: true, emailOnCancel: false };
    }
    return { tab: "PAST", label: "No-show", canCancel: true, emailOnCancel: false };
  }
  return { tab: "PAST", label: String(row.status ?? "Unknown"), canCancel: false, emailOnCancel: false };
};

/**
 * The rows of one tab, grouped by date: [{ date, rows }]. Upcoming runs soonest first; Past and
 * Cancelled run newest day first, with the rows inside a day in time order.
 */
export const bookingGroupsOf = (bookings, tab, now = Date.now()) => {
  const byDate = new Map();
  bookings
    .filter((row) => classifyBooking(row, now).tab === tab)
    .forEach((row) => {
      if (!byDate.has(row.date)) byDate.set(row.date, []);
      byDate.get(row.date).push(row);
    });
  const groups = [...byDate.entries()].map(([date, rows]) => ({
    date,
    rows: [...rows].sort((a, b) => Date.parse(a.startsAt) - Date.parse(b.startsAt)),
  }));
  groups.sort((a, b) => (tab === "UPCOMING" ? a.date.localeCompare(b.date) : b.date.localeCompare(a.date)));
  return groups;
};

/** The confirm message for cancelling one row (section 8.1). */
export const cancelBookingMessage = (row, now = Date.now()) => {
  const { emailOnCancel, label } = classifyBooking(row, now);
  const who = `${row.firstName} ${row.lastName}`.trim();
  const question = `Cancel ${who}'s appointment on ${formatClinicDate(row.date)} at ${row.from}?`;
  if (emailOnCancel) {
    return `${question}\n\nThe slot and the office place become free, and ${row.firstName} gets a cancellation email.`;
  }
  if (label === "No-show") {
    return `${question}\n\nThe appointment time has passed, so no email is sent. The office place becomes free again.`;
  }
  return `${question}\n\nThe appointment has already started, so no cancellation email is sent. The office place becomes free.`;
};

/** The toast for a refused cancel (section 8.2). Every one of these also means "reload". */
export const cancelRefusalMessage = (err) => {
  const status = err?.response?.status;
  const code = err?.response?.data?.errorCode;
  const reason = String(err?.response?.data?.reason ?? "");
  if (status === 404) return "This booking no longer exists. The list has been reloaded.";
  if (status !== 409) return null;
  if (code === "KLE_APT_ALREADY_SCREENED") {
    return "This employee has already been screened, so the booking can't be cancelled. The list has been reloaded.";
  }
  if (reason.startsWith("The booking is already cancelled")) {
    return "Someone else has already cancelled this booking. The list has been reloaded.";
  }
  return "Something changed at the same time. The list has been reloaded. Try again.";
};
```

### 11.3 The new page: `pages/CampaignManagement/Appointments/Bookings.jsx`

A skeleton with the data flow, the tabs and the cancel flow. Take `frame`, `backButton`, `SectionCard`, `RowAction`,
`EmptyState`, `Fact` and the class strings from `Manage.jsx`. The cleanest way is to move them to a shared file, for
example `component/AppointmentCampaigns/ManageParts.jsx`, and import them from both pages.

```jsx
import React, { useCallback, useEffect, useMemo, useState } from "react";
import { useRouter } from "next/router";
import Loader from "../../../component/Loader";
import ConfirmModal from "../../../component/Modal";
import { showToast } from "../../../component/Toaster";
import {
  frame, SectionCard, RowAction, EmptyState, Fact, headerCell,
} from "../../../component/AppointmentCampaigns/ManageParts"; // moved out of Manage.jsx, see above
import {
  cancelAppointmentBookingApi,
  getAppointmentCampaignApi,
  listOfficeBookingsApi,
} from "../../../API/AllApi";
import {
  adminMessage,
  BOOKING_TABS,
  BOOKING_TAB_LABELS,
  bookingGroupsOf,
  cancelBookingMessage,
  cancelRefusalMessage,
  classifyBooking,
  formatClinicDate,
  formatDateRange,
  formatPlainDate,
} from "../../../Utils/appointmentCampaigns";

const MINUTE = 60 * 1000;

const OfficeBookings = () => {
  const router = useRouter();
  const { campaignKey, officeKey, clientType, clientId, clientName } = router.query;

  const [campaign, setCampaign] = useState(null);
  const [bookings, setBookings] = useState([]);
  const [loading, setLoading] = useState(true);
  const [problem, setProblem] = useState(null); // { kind: "missing" | "campaignGone" | "officeGone" | "error", message? }
  const [tab, setTab] = useState("UPCOMING");
  const [now, setNow] = useState(Date.now());
  const [toCancel, setToCancel] = useState(null);
  const [cancelling, setCancelling] = useState(false);

  // Rows move from Upcoming to Past as time passes (section 7).
  useEffect(() => {
    const timer = setInterval(() => setNow(Date.now()), MINUTE);
    return () => clearInterval(timer);
  }, []);

  const load = useCallback(async () => {
    if (!campaignKey || !officeKey) {
      setProblem({ kind: "missing" });
      setLoading(false);
      return;
    }
    try {
      const [{ data: campaignData }, { data: bookingData }] = await Promise.all([
        getAppointmentCampaignApi(campaignKey),
        listOfficeBookingsApi(campaignKey, officeKey),
      ]);
      if (!(campaignData?.offices ?? []).some((office) => office.key === officeKey)) {
        setProblem({ kind: "officeGone" });
      } else {
        setCampaign(campaignData);
        setBookings(Array.isArray(bookingData) ? bookingData : []);
        setProblem(null);
      }
    } catch (err) {
      if (err?.response?.status === 404) {
        const reason = String(err?.response?.data?.reason ?? "");
        setProblem({ kind: reason.startsWith("Office") ? "officeGone" : "campaignGone" });
      } else {
        setProblem({ kind: "error", message: adminMessage(err, "Could not load the bookings.") });
      }
    } finally {
      setLoading(false);
      setNow(Date.now());
    }
  }, [campaignKey, officeKey]);

  useEffect(() => {
    if (router.isReady) load();
  }, [router.isReady, load]);

  const office = campaign?.offices?.find((o) => o.key === officeKey);
  const counts = useMemo(() => {
    const result = { UPCOMING: 0, PAST: 0, CANCELLED: 0 };
    bookings.forEach((row) => { result[classifyBooking(row, now).tab] += 1; });
    return result;
  }, [bookings, now]);
  const groups = useMemo(() => bookingGroupsOf(bookings, tab, now), [bookings, tab, now]);

  const backToCampaign = () =>
    router.replace({
      pathname: "/CampaignManagement/Appointments/Manage",
      query: { campaignKey, clientType, clientId, clientName },
    });

  const confirmCancel = async () => {
    if (!toCancel || cancelling) return; // the modal keeps its buttons live while busy
    setCancelling(true);
    try {
      await cancelAppointmentBookingApi(toCancel.reference);
      showToast("Success", "Booking cancelled.");
    } catch (err) {
      if (err?.response?.status === 403) showToast("Error", "You don't have permission to do that.");
      else showToast("Error", cancelRefusalMessage(err) ?? adminMessage(err));
    } finally {
      await load(); // after success and after every refusal (section 8.2)
      setCancelling(false);
      setToCancel(null);
    }
  };

  if (loading) return frame(<div className="flex justify-center py-20 bg-white rounded-2xl shadow-md"><Loader /></div>);

  if (problem) {
    // Render the error card for problem.kind as in section 4.3: the "missing", "campaignGone" and
    // "error" cards copy Manage.jsx; "officeGone" adds a "Back to the campaign" button → backToCampaign().
  }

  return frame(
    <>
      {/* Header card: {office.name}: bookings, the campaign line, then Places / Employees / Clinic dates */}
      {/* <Fact label="Places">{office.placesUsed} used of {office.workforceSize} ({office.placesLeft} left)</Fact> */}
      {/* <Fact label="Clinic dates">{formatDateRange(campaign.firstSessionDate, campaign.lastSessionDate)}</Fact> */}

      <SectionCard icon="fa-users" title="Bookings" count={bookings.length}>
        <div role="tablist" className="flex gap-2 mb-6">
          {BOOKING_TABS.map((key) => (
            <button
              key={key}
              type="button"
              role="tab"
              aria-selected={tab === key}
              onClick={() => setTab(key)}
              className={`px-4 py-2 rounded-xl text-sm font-semibold border transition-all ${
                tab === key
                  ? "bg-[#152e4c] text-white border-[#152e4c]"
                  : "bg-white text-[#152e4c] border-gray-200 hover:border-[#152e4c]"
              }`}
            >
              {BOOKING_TAB_LABELS[key]} ({counts[key]})
            </button>
          ))}
        </div>

        {groups.length === 0 ? (
          <EmptyState icon="fa-users" title={`No ${BOOKING_TAB_LABELS[tab].toLowerCase()} bookings`} text="…" />
        ) : (
          groups.map((group) => (
            <div key={group.date} className="mb-6">
              <div className="flex justify-between items-baseline mb-2">
                <h4 className="font-bold text-[#152e4c]">{formatClinicDate(group.date)}</h4>
                <span className="text-xs text-gray-500">
                  {group.rows.length} {group.rows.length === 1 ? "booking" : "bookings"}
                </span>
              </div>
              <div className="overflow-x-auto border border-gray-100 rounded-xl">
                <table className="w-full text-sm">
                  <thead>
                    <tr className="bg-gray-50 border-b border-gray-100 text-left">
                      <th className={headerCell}>Time</th>
                      <th className={`${headerCell} w-full`}>Name</th>
                      <th className={headerCell}>Email</th>
                      <th className={headerCell}>Born</th>
                      <th className={headerCell}>Sex</th>
                      <th className={headerCell}>Status</th>
                      <th className={`${headerCell} text-right`}>Action</th>
                    </tr>
                  </thead>
                  <tbody>
                    {group.rows.map((row) => {
                      const { label, canCancel } = classifyBooking(row, now);
                      return (
                        <tr key={row.reference} className="border-b border-gray-100 last:border-0">
                          <td className="py-4 px-4 whitespace-nowrap">{row.from}–{row.to}</td>
                          <td className="py-4 px-4">
                            <div className="font-semibold text-gray-800">{row.firstName} {row.lastName}</div>
                            <div className="text-xs text-gray-400 select-all" title={row.reference}>
                              ref {row.reference.slice(0, 8)}…
                            </div>
                          </td>
                          <td className="py-4 px-4 whitespace-nowrap">{row.email}</td>
                          <td className="py-4 px-4 whitespace-nowrap">{formatPlainDate(row.dateOfBirth)}</td>
                          <td className="py-4 px-4">{row.sex === "FEMALE" ? "F" : row.sex === "MALE" ? "M" : "—"}</td>
                          <td className="py-4 px-4 whitespace-nowrap">{label}</td>
                          <td className="py-4 px-4 text-right">
                            {canCancel && (
                              <RowAction icon="fa-times-circle" label="Cancel" tone="danger"
                                         onClick={() => { setNow(Date.now()); setToCancel(row); }} />
                            )}
                          </td>
                        </tr>
                      );
                    })}
                  </tbody>
                </table>
              </div>
            </div>
          ))
        )}
      </SectionCard>

      <ConfirmModal
        isOpen={!!toCancel}
        message={toCancel ? cancelBookingMessage(toCancel, now) : ""}
        confirmText="Cancel booking"
        cancelText="Keep booking"
        isLoading={cancelling}
        onConfirm={confirmCancel}
        onCancel={() => !cancelling && setToCancel(null)}
      />
    </>,
  );
};

export default OfficeBookings;
```

### 11.4 Manage page changes: `pages/CampaignManagement/Appointments/Manage.jsx`

**The Bookings action on the office row.** Add it first, in the `offices.map(...)` actions cell:

```jsx
<div className="flex gap-2 justify-end">
  <RowAction
    icon="fa-users"
    label="Bookings"
    title="See and cancel this office's bookings"
    onClick={() =>
      router.push({
        pathname: "/CampaignManagement/Appointments/Bookings",
        query: {
          campaignKey: campaign.key,
          officeKey: office.key,
          clientType: router.query.clientType,
          clientId: router.query.clientId,
          clientName,
        },
      })
    }
  />
  <RowAction icon="fa-pen" label="Edit" onClick={() => openDialog("office", office.key)} />
  {/* existing Remove … */}
</div>
```

**The disabled Remove tooltip on a booked clinic day** (section 6):

```jsx
title={
  ended
    ? "This clinic day has ended"
    : booked
      ? "It has bookings. Cancel the upcoming ones on the office's Bookings page. A day with screened bookings can't be removed."
      : "Remove this clinic day"
}
```

### 11.5 Tests worth writing

`classifyBooking` is the piece that is easiest to get wrong. Unit-test it with a fixed `now` for all six rules of
section 7, including:

- `COMPLETED` with a future `startsAt` → Past / Screened;
- `ACTIVE` exactly at `startsAt` → in progress, no email;
- `ACTIVE` exactly at `endsAt` → No-show;
- `CANCELLED` + `ADMIN` with `cancelledByAdmin: null` → still renders.

---

## 12. Not in scope

- **Cancel every booking on a clinic day** (bulk cancel): rejected for now (D142).
- **Phone number, questionnaire answers, consent**: deliberately not sent (D143).
- **Paging and backend filters**: none (D144).
- **People in the clinic-day slot panel**: no (D141).
- **A "Bookings" shortcut on the clinic-day row**: not decided. It's a frontend choice; if you add one, it opens the
  same page scrolled to that `date`.
- **Changing a booking's slot as the admin**: not available. Only the employee can change it, through their manage
  link.

---

## 13. Checklist before you open the PR

- [ ] The office row has a **Bookings** action, first, always enabled, routing with `campaignKey`, `officeKey`,
      `clientType`, `clientId` and `clientName`.
- [ ] The page loads the campaign and the bookings together, and handles a missing key, a gone campaign, a gone
      office and a general error (section 4.3).
- [ ] Rows are split into tabs **only** by `classifyBooking` (section 7): `COMPLETED` before the clock, and instants
      compared with `Date.parse`, not strings.
- [ ] Tabs are recomputed at least once a minute, and just before the confirm modal opens.
- [ ] Rows are grouped by `date`, Upcoming soonest first, Past and Cancelled newest day first.
- [ ] Cancel appears only on `ACTIVE` rows. The confirm wording depends on rule 4 / 5 / 6.
- [ ] After a cancel: reload **both** calls, on success and on every refusal. `KLE_APT_ALREADY_SCREENED` is shown
      with our own wording, not the API's `reason`.
- [ ] `dateOfBirth` and `date` are formatted as plain dates; `cancelledAt` is shown in UK time.
- [ ] No phone column, and nothing asks for one.
- [ ] The Remove tooltip on a booked clinic day is updated.
- [ ] `classifyBooking` has unit tests for all six rules.
