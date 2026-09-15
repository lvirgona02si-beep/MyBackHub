# Scoliosis Masterclass funnel

Two pages, hand-written, no build step. Drop straight into GoHighLevel or any
static host.

| File | Replaces |
|---|---|
| `registration.html` | `watch.mybackhub.com/sc-masterclass-registration` |
| `confirmation.html` | `watch.mybackhub.com/thank-you-sc-masterclass` |
| `live.html` | New. There is no equivalent on the live funnel. |

Structure and copy rhythm follow the Golden Key workshop funnel. See
[../../docs/source-funnels.md](../../docs/source-funnels.md).

## Registration page

- Form sits above the fold beside the headline. The live page renders a
  permanently broken `Getting sessions...` state where the session picker
  should be, which is the single biggest conversion problem on it.
- Headline interrupts a decision in progress rather than describing a benefit.
- Six specific discovery bullets replacing three abstract ones.
- Session countdown, stat strip, and a "Worth Your Time If" qualification block.
- Six video testimonials replacing the Elfsight text widget. Mapping in
  [media/MANIFEST.md](media/MANIFEST.md).
- Medical disclaimer, and a sticky call-to-action bar on mobile.

## Confirmation page

- Live session countdown with a progress bar, flipping to an open/join state
  at zero. The current page never tells anyone when their session is.
- Google, Outlook and `.ics` calendar links generated from the session time.
- Four-item checklist rewritten for an exercise-based session.
- Post-session "Book A Call" call-to-action.
- Dr. Mike's welcome video carried across from the current page.

## Live room

The waiting room and session player. Modelled on Golden Key's `/uk-workshop/live`.

- Continues the countdown the confirmation page started rather than beginning a
  second one. The start time is written to both `sessionStorage` and
  `localStorage` under `mbhMasterclassStart`, because someone clicking the
  joining link from their email arrives in a fresh browser session.
- Late arrivals are dropped into the session already in progress, the way they
  would be on a real live call. Past `LATE_JOIN_MAX_MS` they have missed too
  much to follow it and get a fresh session from the top.
- Starts muted with a "Tap for sound" prompt, because browsers only permit
  autoplay while muted and the click that got them here does not carry across
  the page load.
- Host chat feed timed to seconds watched, so it stays in step whether someone
  joins late or switches tabs.
- Watch-progress milestones at 25 / 50 / 75 / 95 percent, fired to a Meta pixel
  if one is present and to a GHL inbound webhook.

**The chat is host-only by design.** There are no scripted attendee messages.
Inventing patients who say a treatment worked is not something to ship on a
medical offer. Real questions go to the live Q&A at the end.

### Three constants to set

At the top of the script in `live.html`:

| Constant | Currently | Set to |
|---|---|---|
| `MASTERCLASS_SRC` | Dr. Mike's welcome video, as a testable stand-in | The real masterclass recording |
| `MASTERCLASS_SECONDS` | `99`, the stand-in's length | The real runtime in seconds |

The recording is still being produced. When it lands, follow
[SWAP-IN-REAL-VIDEO.md](SWAP-IN-REAL-VIDEO.md): two constants change, the
progress marks and late-join cap recalculate themselves, and `HOST_MESSAGES`
needs retiming against the actual recording.
| `GHL_PROGRESS_WEBHOOK` | Set | Done |

### Progress payload

`GHL_PROGRESS_WEBHOOK` fires one POST per milestone. The workflow behind it
only ever receives this shape, so it cannot be triggered by anything else on
the funnel:

```json
{
  "type": "masterclass_progress",
  "tag": "watched-50",
  "percent": 50,
  "seconds_watched": 1050,
  "watched_at": "2026-09-15T19:30:00.000Z",
  "contact_id": "<from ?c= on the joining link>",
  "email": "<if known>",
  "first_name": "<if known>"
}
```

`tag` arrives ready-made (`watched-start`, `watched-25`, `watched-50`,
`watched-75`, `watched-complete`) so the workflow applies it rather than
branching on a percentage.

Nothing is sent unless a `contact_id` or `email` is known. An empty email
reaching a Create/Update Contact action matches nothing and creates a blank
record.

The joining link in your email needs `?c={{contact.id}}` on it, or attendees
arrive unidentified and no progress is recorded.

## Before launch

- [ ] Wire `#regForm` to the LeadConnector/GHL endpoint and repoint the success
      state at the real session room. It is front-end only right now.
- [ ] Set both countdowns to the true session cadence. They currently roll to
      the next `:00` or `:30`.
- [ ] Set `LIVE_ROOM_URL` in `confirmation.html` if the waiting room is not at
      `live.html`, and repoint Book A Call and the footer nav on every page.
- [ ] Set the three `live.html` constants above.
- [ ] Remove the build note from the footer of each page.
- [ ] Reconcile brand tokens against the brand guidelines.
      See [../../docs/brand-tokens.md](../../docs/brand-tokens.md).

## Form fields

First name, last name, email, mobile, consent checkbox. Nothing else. Every
extra field costs completions, and anything about the person's curve is a
better conversation for the call than the opt-in.
