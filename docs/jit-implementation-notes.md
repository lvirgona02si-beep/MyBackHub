# JIT funnel: how this build differs from the reference

[just-in-time-webinar-logic.md](just-in-time-webinar-logic.md) is the original
build reference. The fuller master spec is
[SIMULATED-LIVE-WEBINAR-SPEC.md](../SIMULATED-LIVE-WEBINAR-SPEC.md) at the repo
root. This records where the scoliosis masterclass diverges from them, and why.

## Constants

| Constant | Reference | Here | Why |
|---|---|---|---|
| Session length | 30 min | 37:48 (2268s) | Dr. Mike's session, plus Q&A |
| `SLOT_MS` | 15 min | 15 min | Same |
| `MIN_LEAD_MS` | 3 min | 3 min | Same |
| `OFFSETS_MS` | 1h / 3h / 6h | 1h / 3h / 6h | All clear the 37:48 minimum gap |
| `START_HOUR` | 9 | 9 | Same |
| `CUTOFF_HOUR` | 21 | 21 | Same |
| `ROLLOVER_HOURS` | 9 / 12 / 17 | 9 / 12 / 17 | Same |
| Complete threshold | 90% | 90% | Was 95%, now matches the spec |
| Storage prefix | `gk*` | `mbh*` | Prefixed per the collision trap |

Verified by running the real `laterSlots` source at **72 times of day**: the
soonest slot always lands 3 to 18 minutes out on a `:00`/`:15`/`:30`/`:45`
boundary, later slots only on `:00`/`:30` and only inside 9am to 9pm, no two
offered sessions are ever closer than the 37:48 runtime, and the dropdown is
never left with fewer than two options.

Two bugs that sweep caught and that are worth not reintroducing:

- **Offsets before `START_HOUR` were rolling to tomorrow.** At 6am the list read
  "Tomorrow at 9:00 am" while today's 9am was still three hours away. The
  fallback queue now runs today's remaining published hours before tomorrow's.
- **A substituted fallback could land 30 minutes from another offered slot.**
  The overlap check only measured against the soonest session, not against the
  others already chosen. Two sessions half an hour apart cannot both run a
  37 minute masterclass, and offering both reads as a recording.

## Deliberate divergences

**The Wistia media id is not in a `video.js`.** The spec keeps every media id in
one shared file because five pages read them. This funnel has exactly one page
that plays video, and [conventions.md](conventions.md) says a page is one
self-contained file. The id lives in `live.html`'s CONFIGURE ME block instead.
If an on-demand or recap page is ever added, lift it into `video.js` and have
all three read from it rather than copying the id.

**No on-demand, recap or booking page.** The spec's shape has `watch.html`,
`recap.html` and `booking.html`. This funnel sells a $99 checkout off the back
of the masterclass rather than booking a call, so `checkout.html` is the
equivalent of `booking.html`, and the other two are out of scope until there is
a second cut of the video to put on them.

**No mirror page.** The spec duplicates the room at a meaningless path because
Australian carriers silently drop SMS whose links read as marketing
infrastructure. This funnel's `MARKET` is the United States. Add one before any
Australian SMS send, and keep the two byte-identical.

**The chat has a composer, and it posts nowhere.** The earlier build here had no
input at all, on the grounds that a box that visibly goes nowhere is its own
kind of misleading. The spec's argument wins: the recording asks people more
than once to put an answer in the chat, and a prompt with no input is worse than
no prompt. What is typed is rendered locally with `textContent` and sent
nowhere.

**Simulated attendees are wired but switched off.** `SIM_ON` is `false` and
`SIM_MESSAGES` is empty, pending the cast being written by hand. The constraint
recorded against that work: this is a medical offer, so an invented attendee
reporting that a treatment worked is a fabricated patient testimonial, which is
an FTC problem before it is a taste problem.

## Fixes ported from the Golden Key handover, 22 Sep 2026

Three faults found by live testing of the reference funnel *after* the spec was
written, so this build had inherited all three.

**A booking further out than one slot is handed back to the holding page.** The
room is exempt from the plausibility guard for a booked start, which is right,
but it had no state for a session that is not about to begin: it went straight
to a countdown in front of a player it had no reason to open, and a next-day
booking rendered as `Starting in 1092:04`. Both pages use the same threshold
(`SLOT_MS + MIN_LEAD_MS`, 18 minutes) and `?c=` and `?t=` travel with the
redirect. `?in=` is exempt, being capped at an hour and meant to be watched from
the room. Both countdown formatters also grew an hour branch as a backstop.

No loop is possible, and it is checked rather than assumed: the holding page
only offers a way into the room once its own target has arrived, by which point
the room's redirect condition is false. Verified minute by minute from -5 to
+1200.

**The holding page reads `?t=` off the link.** Storage belongs to the browser
that registered. Someone who booked tomorrow morning and opened the page on a
second device had nothing to find, and the page read that as the booking being
*lost* rather than as it being somewhere else, falling through to the branch
that opens the session immediately. They were shown a live badge and a join
button for a session hours away. Same guards as the room: ISO or epoch, ignored
beyond a day either side, ignored outright if the merge field arrived
unresolved. Storage remains the fallback, so a link without `?t=` behaves as
before.

**Known limit, unchanged:** a second-device visitor arriving with *no* `?t=` at
all still gets the wrong screen. Nothing on a static page can fix that. There is
no booking to read and no API key to look one up with.

**The ICS was silently truncating.** `DESCRIPTION:` ran to 162 octets against a
75 octet cap, past which a calendar app drops the rest of the line rather than
erroring. `icsFold()` folds content lines with a single leading space on each
continuation, counted in octets and stepped by code point so a surrogate pair is
never split. `icsEscape()` escapes the ICS grammar characters, which matters
here because the event title legitimately contains a comma.

The joining link stays out of the calendar entry deliberately. The room link
ends `?c={{contact.id}}` and the entry is built in the browser, where there is
no merge field to resolve, so any embedded link would be the generic one and the
attendee arriving on it would be anonymous to the room. The invite says where
the real link is instead.

Two gaps this surfaced on our build and fixed at the same time: the booked card
said "today" for a next-day session, and a booked registrant returning at their
session time landed on that card with no way through.

## Open decisions

**The Q&A.** Registration, confirmation and the room all describe a live Q&A
with Dr. Mike at the end, carried over from the current MyBackHub copy. That
claim is true only if someone is genuinely on the other end. If the session is a
recording start to finish, this is the one place the build crosses the line the
reference draws, and the copy needs changing rather than the code. Decide before
launch.

**Sessions on hold.** `SESSION_ON_HOLD` is wired on the confirmation page and
the live room, and deliberately absent from registration, so opt-in rate stays
comparable between hold days and normal days.

## Testing the webhooks

The endpoint itself is fine. Verified directly: `OPTIONS` returns 204 with
`access-control-allow-origin: *` and POST permitted, and a `POST` of the
progress payload returns `200`. If nothing is arriving, it is one of these.

**1. You are testing in the Artifact preview.** That sandbox blocks all outbound
`fetch` to third-party hosts, with no visible error. The webhook can never fire
there. Test on a real host: GitHub Pages, or the page in GHL.

**2. Nothing identifies the viewer.** `pingProgress` refuses to send without a
`contact_id` or an `email`, because an empty identifier reaching a
Create/Update Contact action creates a blank record. Opening `live.html`
directly gives you neither. Add `?c=TEST123` to the URL, or arrive through the
confirmation page after registering.

**3. No milestone has been reached.** `watched-start` fires on open, but the
others need 25/50/75/90 percent of the runtime in **played** seconds. Progress
now reads the player's own `currentTime` rather than time on the page, so a
stalled connection or a phone waiting for a tap accrues nothing.

`DEBUG_PINGS` is on in `live.html`. Open the console: every send and every skip
is logged with the reason. Turn it off once the funnel is verified.

## Still needed

- `HOST_MESSAGES` timings, anchored to the caption track. See
  [SWAP-IN-REAL-VIDEO.md](../funnels/scoliosis-masterclass/SWAP-IN-REAL-VIDEO.md).
- `SIM_MESSAGES`, if the client wants a populated room.
- Meta pixel base code on all four pages. `fbq` is called only if already
  defined, so nothing breaks in the meantime and the Meta fields in the
  registration payload stay empty strings.
- `Lead` and `Schedule` must fire server-side from the CRM Conversions API,
  passing the `event_id` from the registration payload. Neither is fired from
  the browser here, on purpose.
- `session_time_iso` must exist as a GHL contact field **and** be mapped from
  the inbound webhook, or `&t=` on the joining link resolves to nothing and the
  room silently reverts to opening the moment it loads.
- `?c={{contact.id}}` and `&t={{contact.session_time_iso}}` on the joining link
  in your emails. Both are carried onward from there, but they have to start
  somewhere. Send yourself a test and click it: the address bar has to show a
  real timestamp, not `%7B%7B…`.
