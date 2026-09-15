# Just-in-Time Webinar Funnel — How It Works

A build reference for replicating this funnel on a different offer.

Everything here is client-side JavaScript on three static HTML pages. There is
no server, no database and no framework. That is the point: it runs on GitHub
Pages, and the only backend is the CRM's inbound webhook.

**Source files this describes:**
`uk-workshop/index.html` (registration) · `uk-workshop/confirmation.html`
(holding) · `uk-workshop/live.html` (the room)

---

## 1. The premise

A recorded workshop that behaves like a live one. Sessions appear to run on a
rolling schedule; the visitor picks the next one, waits a few minutes, and the
video opens and plays by itself at the appointed time. Arriving late drops them
into a session already in progress rather than restarting it.

Nothing is actually scheduled. Every "session" is computed from the visitor's
own clock at the moment they land. The scheduling is an illusion held together
by three rules:

1. Start times always round to a clean clock time, never "in 4 minutes 38 seconds".
2. The chosen start time is **persisted**, so it never moves once picked.
3. Lateness is honoured, not erased — the video seeks forward.

Rule 2 is the one people get wrong. If the start time is recomputed on each page,
the countdown jumps and the illusion dies instantly.

---

## 2. The three pages and what each owns

| Page | Owns | Writes | Reads |
|---|---|---|---|
| **Registration** | Choosing the slot, capturing the lead | `gkWorkshopStart`, `gkWorkshopChoice`, `gkAttendee` | — |
| **Confirmation** | Counting down, calendar links | — | all three |
| **Live** | Playing, tracking, chat | `gkWorkshopStart` (only if written off) | all three |

The handover between pages is **storage, not URL parameters**. A refresh, a
back button or a reopened tab all land in the same state.

---

## 3. Storage model

Four keys. Each is written to **both** `sessionStorage` and `localStorage`, and
read from `sessionStorage` first with `localStorage` as fallback.

```js
const SESSION_KEY = 'gkWorkshopStart';    // epoch ms of the chosen start
const CHOICE_KEY  = 'gkWorkshopChoice';   // {jit: bool, at: epoch ms}
// 'gkAttendee'    — {first_name, email, contact_id}
// 'gkOriginPage'  — where "back to homepage" should point
```

```js
function readStart() {
  let v = 0;
  try { v = parseInt(sessionStorage.getItem(SESSION_KEY) || '0', 10) || 0; } catch (e) {}
  if (!v) { try { v = parseInt(localStorage.getItem(SESSION_KEY) || '0', 10) || 0; } catch (e) {} }
  return v;
}
function rememberStart(t) {
  try { sessionStorage.setItem(SESSION_KEY, String(t)); } catch (e) {}
  try { localStorage.setItem(SESSION_KEY, String(t)); } catch (e) {}
}
```

**Why both.** `sessionStorage` is the right lifetime for a single sitting and dies
with the tab. But someone who clicks the reminder email an hour later arrives in
a *fresh* session where `sessionStorage` is already empty — without the
`localStorage` mirror they look like a brand new visitor and get handed a second
countdown for a session they already booked.

**Why every access is wrapped in try/catch.** Private windows and blocked site
data make these accessors *throw*, not return null. One unguarded read takes the
whole page down. Assume storage can fail at any moment and still render.

---

## 4. Choosing the session (registration page)

### The soonest slot

```js
const SLOT_MS     = 15 * 60 * 1000;   // sessions land on :00/:15/:30/:45
const MIN_LEAD_MS = 3 * 60 * 1000;    // never less than 3 min away

function workshopStartAt() {
  const now = Date.now();
  let target = readStart();
  if (!target || target <= now || target - now > SLOT_MS + MIN_LEAD_MS) {
    target = Math.ceil(now / SLOT_MS) * SLOT_MS;       // next quarter hour
    if (target - now < MIN_LEAD_MS) target += SLOT_MS; // too close to register
    rememberStart(target);
  }
  return target;
}
```

Three things worth copying exactly:

- **`Math.ceil(now / SLOT_MS) * SLOT_MS`** is the whole trick. Round the current
  time up to the next slot boundary and you get a real clock time for free.
- **`MIN_LEAD_MS`** stops the absurd case: landing at 2:59 and being offered a
  3:00 session you cannot possibly fill the form for. Push to the next slot.
- **The stored value wins** unless it is in the past or implausibly far ahead.
  This is rule 2 from section 1, and it is the line that keeps the countdown
  stable across pages.

### The later slots

```js
const OFFSETS_MS  = [60 * 60000, 3 * 3600000, 6 * 3600000];
const CUTOFF_HOUR = 21;   // nothing scheduled past 9pm local
```

Offsets from *now*, rounded up, same-day only, deduplicated against the soonest
slot so the same time is never offered twice.

**Two rules learned the hard way:**

1. **No offset shorter than the session length.** A 30-minute workshop cannot
   start 15 minutes after the previous one. Offering both advertises that the
   times are generated. Minimum gap ≥ session duration.
2. **Later slots round to `:00`/`:30`, not `:15`/`:45`.** Published timetables do
   not run at quarter past. Quarter-hour starts are the tell that the times came
   from the visitor's own clock. The *soonest* slot keeps finer rounding, because
   it is sold as "starting shortly" rather than as a scheduled time.

```js
function roundUpTo30(d) {
  const r = new Date(d);
  r.setSeconds(0, 0);
  const m = r.getMinutes();
  r.setMinutes(m + ((30 - (m % 30)) % 30));
  return r;
}
```

---

## 5. Capturing the lead

On submit, in this order:

1. Write `gkAttendee`, `gkWorkshopChoice`, and `gkWorkshopStart` if just-in-time.
2. Gather Meta match signals (below).
3. `POST` to the CRM inbound webhook.
4. **Only on a successful response**, show success and redirect after ~1.8s.

```js
const fbclid = new URLSearchParams(window.location.search).get('fbclid') || '';
let fbc = getCookie('_fbc');
if (!fbc && fbclid) fbc = 'fb.1.' + Date.now() + '.' + fbclid;
const fbp = getCookie('_fbp');
```

`_fbc` is only set by Meta's script once it has seen `fbclid`. On a first
pageview it is often not written yet, so **synthesise it from the `fbclid` in the
URL** using Meta's documented `fb.1.<timestamp>.<fbclid>` format. Skip this and
attribution quality drops sharply on exactly the traffic you paid for.

**Registration payload:**

```json
{
  "type": "workshop_registration",
  "first_name": "...", "last_name": "...", "email": "...", "phone": "...",
  "session_time_iso": "...", "session_is_just_in_time": true,
  "country": "United Kingdom", "market": "United Kingdom",
  "funnel_type": "Workshop",
  "event_id": "...", "fbclid": "...", "fbp": "...", "fbc": "...",
  "user_agent": "..."
}
```

`type` exists so one webhook can carry several event kinds and the workflow
branches on it with an If/Else.

**Never redirect on a failed webhook.** Re-enable the button and show an error.
A visitor who lands on the confirmation page without a CRM record is invisible —
no reminder, no follow-up, and no way to find out it happened.

---

## 6. The countdown (confirmation page)

Reads the stored start and ticks once a second. Two display states:

- **Just-in-time** → countdown, swapping to a join button at zero
- **Fixed later slot** → the booked card with Google / Outlook / `.ics` links

Calendar links are built inline. The `.ics` is a `data:` URL with a `download`
attribute — no server needed.

The join button links to the live page. It is **not** an auto-redirect: reaching
the room should be a deliberate click, so the player only ever starts on a page
the visitor chose to open.

---

## 7. The live room

### Deciding where to start

```js
const now = Date.now();
let target = readStart();
if (target - now > SLOT_MS + MIN_LEAD_MS) target = 0;                          // stale future
if (!target || now - target > LATE_JOIN_MAX_MS) { target = now; rememberStart(target); }
if (target <= now) { reveal((now - target) / 1000); return; }                  // late → join in progress
// otherwise: run the countdown, then reveal()
```

Four cases, in order:

| State | Behaviour |
|---|---|
| Start time implausibly far ahead | Treat as stale, discard |
| Nothing stored at all | Open **now** — they lost the booking, don't make them wait twice |
| Past the late cap | Write the session off, fresh start from zero |
| In the past, within the cap | `reveal(offset)` — join already running |

### The late-join cap

```js
const LATE_JOIN_MAX_MS = Math.min(10 * 60, WORKSHOP_SECONDS) * 1000;
```

Ten minutes, **or the session length if that is shorter**. The second clause
matters: joining 12 minutes into a 10-minute video would land past the end.

> **Gotcha that will bite you in testing.** While you are using a short stand-in
> video, `WORKSHOP_SECONDS` is small, so the cap is small too. With a 90-second
> placeholder, arriving 2 minutes late gives a *fresh start*, not a 2-minute
> offset — and it looks like the late-join feature is broken when it is working
> correctly. Late-joining can only be tested properly once the real duration
> is in.

### Seeking the player

```js
f.src = WORKSHOP_SRC + '&muted=' + (muted ? 1 : 0) + '&t=' + Math.max(0, Math.round(startAt));
```

The `&t=` start-offset parameter is what makes late joining work. Whatever player
you use must support it — Tella, YouTube, Vimeo all do. **Check this before
choosing a host**, because without it the whole premise collapses.

### The muted-autoplay problem

Browsers only permit autoplay while muted, and the click that got them here
happened on the *previous page*, so it does not carry as a user gesture.

So: open **muted and running**, with a prominent tap-for-sound bar. Tapping
rebuilds the iframe unmuted — at the position they had reached, not from the top:

```js
sessionOpenedAt = Date.now() - offsetSec * 1000;   // wall-clock time of video position 0
// on tap:
buildPlayer(false, (Date.now() - sessionOpenedAt) / 1000);
```

Storing the wall-clock time of position zero, rather than a counter, means the
rebuild lands correctly no matter when they tap.

---

## 8. Watch tracking

The critical decision: **measure visible time, not wall-clock time.**

```js
function engagementTick() {
  const now = Date.now();
  if (document.visibilityState === 'visible') {
    if (lastTick !== null) watchedSeconds += (now - lastTick) / 1000;
    lastTick = now;
  } else {
    lastTick = null;   // paused while the tab is hidden
  }
  // ...
}
setInterval(engagementTick, 1000);
```

A backgrounded tab accrues nothing. Without this, someone who opens the room and
walks away gets tagged as having watched to the end, and your single best signal
for separating a buyer from a browser is worthless.

Milestones fire once each:

```js
const MARKS = [
  { at: 0.25, ev: 'Workshop25',       tag: 'watched-25',       pct: 25 },
  { at: 0.50, ev: 'Workshop50',       tag: 'watched-50',       pct: 50 },
  { at: 0.75, ev: 'Workshop75',       tag: 'watched-75',       pct: 75 },
  { at: 0.95, ev: 'WorkshopComplete', tag: 'watched-complete', pct: 100 }
];
```

`0.95` rather than `1.0` for completion — nobody sits through the outro, and
holding out for 100% loses you most of the people who actually finished.

### Late arrivals and skipped marks

```js
function startEngagementTracking(offsetSec) {
  watchedSeconds = offsetSec;
  for (let i = 0; i < MARKS.length; i++) {
    if (WORKSHOP_SECONDS * MARKS[i].at <= offsetSec) firedMarks[MARKS[i].ev] = true;
  }
  // ...
}
```

Marks the offset skipped past are **recorded as fired but never sent**. They
reached that point of the video; they did not watch it. Tagging them for it
would poison the signal.

**Progress payload** — one per milestone, tag pre-computed so the workflow just
applies it rather than branching on a percentage:

```json
{
  "type": "workshop_progress",
  "tag": "watched-50", "percent": 50,
  "seconds_watched": 912,
  "watched_at": "2026-09-15T14:22:31.004Z",
  "contact_id": "...", "email": "...", "first_name": "..."
}
```

Identifiers are only included **if present**. An empty `email` reaching a
Create/Update Contact action matches nothing and creates a blank record, and you
will be cleaning those out for weeks.

All progress calls are fire-and-forget — `.catch(function () {})` — and must
never block the player.

---

## 9. Identifying the viewer

```js
let cid = new URLSearchParams(window.location.search).get('c');
if (cid && /[{}]/.test(decodeURIComponent(cid))) cid = null;
```

Email links carry `?c={{contact.id}}`. Two things:

- **Carry it through** to any onward link, so identity survives the hop.
- **Reject unsubstituted merge fields.** If the email editor fails to substitute,
  a literal `{{contact.id}}` arrives and would be sent to the CRM as an
  identifier matching nothing. The brace test is two lines and saves a confusing
  mess later.

---

## 10. The simulated chat

An array of messages with timestamps in seconds, released against
`watchedSeconds`:

```js
function releaseDueMessages(seconds) {
  for (let i = 0; i < HOST_MESSAGES.length; i++) {
    const m = HOST_MESSAGES[i];
    if (!firedMessages[i] && seconds >= m.at) {
      firedMessages[i] = true;
      addMessage(m.from, m.text, { link: m.link });
    }
  }
}
```

Because it keys off `watchedSeconds` and that is seeded with the join offset, a
late arrival sees the backlog already there — exactly as on a real call.

**Keep it one-way.** Attendees can send, but nothing they send is broadcast.
Visible fake replies from fake attendees is where this crosses from "recorded
session presented as scheduled" into straightforward deception.

---

## 11. Meta events

| Event | Fired by | Why |
|---|---|---|
| `PageView` | Browser pixel | Landing page views |
| `Lead` | **Server-side**, via CRM Conversions API | Avoids double-counting |
| `Workshop25/50/75/Complete` | Browser, `trackCustom` | Engagement audiences |
| `Schedule` | Server-side on booking | Value-weighted conversion |

The registration `Lead` fires **server-side only**. Firing it from both browser
and CRM double-counts in Ads Manager and corrupts your cost-per-lead, which is
the number you will be optimising against.

---

## 12. Pausing sessions without a rebuild

One boolean, mirrored on the confirmation and live pages:

```js
const SESSION_ON_HOLD = false;
if (SESSION_ON_HOLD) {
  [cdStage, joinStage, bookedStage].forEach(el => { if (el) el.hidden = true; });
  document.getElementById('holdStage').hidden = false;
  document.querySelectorAll('[data-session-only]').forEach(el => { el.hidden = true; });
  // ...rewrite headings and CTA copy...
  return;
}
```

Worth building in from the start. It lets you run the funnel for opt-in-rate
testing before any recording exists, and take sessions down for a day without
touching the ads.

**Critically: it must not touch the registration page.** Leave the pixel, the
webhook and the redirect alone, and your opt-in rate stays comparable between
hold days and normal days.

---

## 13. Replication checklist

**Constants to change**
- [ ] `WORKSHOP_SECONDS` — real duration in seconds. Everything keys off this
- [ ] `WORKSHOP_SRC` — embed URL, must support a start-offset parameter
- [ ] `SLOT_MS`, `MIN_LEAD_MS`, `OFFSETS_MS`, `CUTOFF_HOUR`
- [ ] `HOST_MESSAGES` — chat copy, timed to the new recording
- [ ] Webhook URLs, pixel id, calendar event title
- [ ] Storage key prefix, if the two funnels share a domain — **see below**

**Traps**
- [ ] Storage keys collide if two funnels run on one domain. Prefix them
- [ ] Every storage access in try/catch
- [ ] `MARKS` percentages are of `WORKSHOP_SECONDS` — no hardcoded seconds
- [ ] Late-join cap looks broken while a short placeholder video is in
- [ ] Don't redirect on a failed webhook
- [ ] Don't send empty identifier fields
- [ ] Synthesise `_fbc` from `fbclid` when the cookie is missing
- [ ] Fire `Lead` server-side only

**Test matrix**

| Case | Expected |
|---|---|
| Register, wait | Countdown → player opens muted, plays |
| Register, arrive 2 min late | Joins at 2:00, chat backlog present |
| Arrive past the cap | Fresh session from 0:00 |
| Refresh mid-countdown | Same target, no jump |
| Close tab, reopen from email link | Same session, not a new countdown |
| Private window | Renders and runs; nothing thrown |
| Background the tab 5 min | `watchedSeconds` barely moves |
| Watch to 95% | `watched-complete` fires once |
| `?c={{contact.id}}` literal | Ignored, not sent |

---

## 14. Where the honesty line sits

This presents a recording as a scheduled session. That framing is normal in
this format and defensible, but it survives only on specifics. In this build:

- Sessions are described as "starting shortly", not "live"
- The chat is one-way; no fabricated attendee replies are shown
- Late arrivals genuinely join in progress rather than being told they did
- Nothing claims a presenter is watching in real time

The line to hold: **the scheduling is simulated, the engagement data is real.**
The moment you fake attendee counts, invent chat replies, or claim live
presence, it stops being a format convention and becomes something you would not
want a registrant to see the source of. The technical build above works either
way — which is exactly why the decision has to be made deliberately.
