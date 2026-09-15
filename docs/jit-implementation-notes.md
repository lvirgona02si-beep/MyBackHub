# JIT funnel: how this build differs from the reference

[just-in-time-webinar-logic.md](just-in-time-webinar-logic.md) is the build
reference. This records where the scoliosis masterclass diverges, and why.

## Constants

| Constant | Reference | Here | Why |
|---|---|---|---|
| Session length | 30 min | 35 min | Dr. Mike's session, plus Q&A |
| `SLOT_MS` | 15 min | 15 min | Same |
| `OFFSETS_MS` | 1h / 3h / 6h | 1h / 3h / 6h | All clear the 35 min minimum gap |
| `CUTOFF_HOUR` | 21 | 21 | Same |
| Storage prefix | `gk*` | `mbh*` | Prefixed per the collision trap |

Verified across the day: the soonest slot always lands 3 to 16 minutes out on
a `:00`/`:15`/`:30`/`:45` boundary, later slots only on `:00`/`:30`, and the
smallest gap between any two offered sessions is 45 minutes, comfortably over
the 35 minute session length.

## Deliberate divergences

**Watch tracking reads the video, not the wall clock.** The reference measures
visible time because an embedded player exposes no progress cross-origin. This
build uses a real `<video>`, so progress is its own `currentTime`, which is
accurate rather than estimated. The visibility rule from the reference still
applies and is still enforced: time only accumulates while the tab is in view,
because `currentTime` keeps advancing in a backgrounded tab. Swap in an embed
and you lose this; fall back to the reference's interval counter.

**Unmuting does not rebuild the player.** With an iframe there is no other way
to change the mute parameter. With a `<video>` element, setting `muted = false`
keeps playing from the same frame, so the wall-clock bookkeeping the reference
needs for the rebuild is unnecessary.

**The chat has no input at all.** The reference allows attendees to send
messages that are never broadcast. An input that visibly goes nowhere is its
own kind of misleading, so this build states plainly that questions go to the
live Q&A and offers no box.

## Open decisions

**The Q&A.** Registration, confirmation and the room all describe a live Q&A
with Dr. Mike at the end, carried over from the current MyBackHub copy. That
claim is true only if someone is genuinely on the other end. If the session is
a recording start to finish, this is the one place the build crosses the line
the reference draws in section 14, and the copy needs changing rather than the
code. Decide before launch.

**Sessions on hold.** `SESSION_ON_HOLD` is wired on the confirmation page and
the live room, and deliberately absent from registration, so opt-in rate stays
comparable between hold days and normal days.

## Testing the webhooks

The endpoint itself is fine. Verified directly: `OPTIONS` returns 204 with
`access-control-allow-origin: *` and POST permitted, and a `POST` of the
progress payload returns `200`. If nothing is arriving, it is one of these.

**1. You are testing in the Artifact preview.** The Artifact sandbox blocks all
outbound `fetch` to third-party hosts, with no visible error. The webhook can
never fire there. Test on a real host: GitHub Pages, or the page in GHL.

**2. Nothing identifies the viewer.** `pingProgress` refuses to send without a
`contact_id` or an `email`, because an empty identifier reaching a
Create/Update Contact action creates a blank record. Opening `live.html`
directly gives you neither. Add `?c=TEST123` to the URL, or arrive through the
confirmation page after registering.

**3. No milestone has been reached.** `watched-start` fires on open, but the
others need 25/50/75/95 percent of `MASTERCLASS_SECONDS` in *visible* watched
time. A backgrounded tab accrues nothing, by design.

`DEBUG_PINGS` is on in `live.html`. Open the console: every send and every
skip is logged with the reason. Turn it off once the funnel is verified.

## Still needed

- `GHL_REGISTRATION_WEBHOOK` in `registration.html`. Empty, and the form
  refuses to redirect without it, by design.
- Meta pixel on all three pages. `fbq` is called only if already defined, so
  nothing breaks in the meantime.
- `Lead` and `Schedule` must fire server-side from the CRM Conversions API.
  Neither is fired from the browser here, on purpose.
- `?c={{contact.id}}` on the joining link in your emails. It is carried through
  confirmation to the room to the booking link, but it has to start somewhere.
