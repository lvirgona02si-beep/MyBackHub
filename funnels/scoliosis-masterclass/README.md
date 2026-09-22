# Scoliosis Masterclass funnel

Five pages, hand-written, no build step. Every page is one self-contained
file: open it in a browser and it runs. Drop straight into GoHighLevel or any
static host.

| File | Replaces |
|---|---|
| `registration.html` | `watch.mybackhub.com/sc-masterclass-registration` |
| `confirmation.html` | `watch.mybackhub.com/thank-you-sc-masterclass` |
| `live.html` | New. There is no equivalent on the live funnel. |
| `checkout.html` | `payment.mybackhub.com/the-sc-masterclass-checkout` |
| `thank-you.html` | New. Where the payment link redirects after a sale. |

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

- Resolves the session from `?t=` on the link first, then storage, so someone
  who booked on their phone and opened the page on a laptop sees their booking
  rather than a live badge for a session hours away.
- Live session countdown with a progress bar, flipping to an open/join state
  at zero, and a booked card for anything further out than one slot.
- Five calendar providers generated from the session time: Google, Outlook.com,
  Outlook for work, Yahoo and an `.ics` download for Apple and every desktop
  client. We cannot detect what someone has installed, so all five are always
  offered and only the **order** changes, putting the platform's own calendar
  first. The
  ICS folds its content lines at 75 octets and escapes ICS grammar characters,
  because past that cap a calendar app drops the rest of the line silently.
  The joining link is deliberately not in the entry: it is built in the browser
  where `{{contact.id}}` cannot resolve, so it would be the generic link and the
  attendee arriving on it would be anonymous to the room.
- Four-item checklist rewritten for an exercise-based session.
- Dr. Mike's welcome video carried across from the current page.

## Live room

The waiting room and session player. Modelled on Golden Key's `/uk-workshop/live`.

- Continues the countdown the confirmation page started rather than beginning a
  second one, and resolves which session to open from three sources in order:
  `?t=` on the link, then a booking in `mbhMasterclassChoice`, then the rolling
  slot in `mbhMasterclassStart`. Everything is mirrored to both `sessionStorage`
  and `localStorage`, because someone clicking the joining link from their email
  arrives in a fresh browser session where `sessionStorage` is already gone.
- `?t=` is the only source that survives a change of device, which is the case
  of registering on a phone and opening the room on a laptop. It takes ISO or
  epoch ms, and is ignored if it is more than a day out or contains braces, so
  an unsubstituted `{{contact.session_time_iso}}` arriving literally falls back
  to storage rather than being acted on.
- Late arrivals are dropped into the session already in progress, the way they
  would be on a real live call. Past `LATE_JOIN_MAX_MS` they have missed too
  much to follow it and get a fresh session from the top.
- `mbhMasterclassEntered` tells a **returning** viewer from a late one. A
  browser that has already been inside this session carries on wherever the
  clock is now, however long the tab was shut. Without it, closing the tab and
  reopening restarted the masterclass at zero, and nothing says "recording"
  louder than that.
- `?in=<seconds>` is a demo hook: it starts a countdown of that length, wins
  over everything, resets on reload and is deliberately never persisted, so a
  preview cannot leave a fake booking in the browser of someone who later
  arrives for a real session. It can only ever *delay* the room opening. There
  is no parameter that reveals a session early or skips part of one.
- Starts muted with a "Tap for sound" prompt, because browsers only permit
  autoplay while muted and the click that got them here does not carry across
  the page load. On that tap it plays first and unmutes second, and the bar only
  goes once playback is actually under way.
- Three layers of pause defence: a transparent shield over the video surface,
  long-press suppression so iOS cannot raise its *Save Video / Copy / Share*
  sheet, and a `pause` listener that restarts playback when the phone stops it
  on its own.
- Fullscreen is a CSS overlay on the panel, never `requestFullscreen` on the
  video element. On an iPhone that is the only fullscreen available and it draws
  Apple's own player over the top, scrub bar and all, in the middle of a session
  billed as live. The chat stays on in fullscreen as a strip down the right: a
  reserved column on desktop, and floated over the right-hand edge of the video
  on a phone, where a reserved column would leave the video a sliver.
- A booking further out than one slot is handed back to the holding page, which
  has the state that fits it. The room has nothing to count down to in front of
  a player it has no reason to open, and a next-day booking used to render as
  `Starting in 1092:04`.
- Chat feed timed to **seconds of video watched**, not wall clock, so a line
  lands against the right part of the talk whether someone joined late or
  switched tabs.
- Watch-progress milestones at 25 / 50 / 75 / 90 percent, fired to a Meta pixel
  if one is present and to a GHL inbound webhook. Progress is the player's own
  `currentTime`, taken with `Math.max` so a mark can never un-fire.

**`watched-complete` fires at 90 percent, not 100.** People close the tab in the
last seconds of an outro they have effectively finished, and holding out for the
full length loses most of the audience that actually watched it.

**The attendee chat is wired but switched off.** `SIM_ON` is `false` and
`SIM_MESSAGES` is empty. The machinery is all there, so writing the cast and
flipping the flag is all it takes. The constraint on whoever writes it: this is
a medical offer, and an invented attendee reporting that a treatment worked is
a fabricated patient testimonial, which is an FTC problem before it is a taste
problem. Arrivals, questions and one honest sceptic are the material.

The composer under the chat renders what is typed locally and posts it nowhere.

### Constants to set

At the top of the script in `live.html`:

| Constant | Currently | Notes |
|---|---|---|
| `WISTIA_ID` | `5i1tmdo2w2` | Set. The only place the media id is written |
| `MASTERCLASS_SECONDS` | `2268` | Overwritten by Wistia's own duration on load, so a re-cut needs no change |
| `GHL_PROGRESS_WEBHOOK` | Set | Done |
| `SESSION_ON_HOLD` | `false` | Flip to take sessions down for a day |
| `HOST_MESSAGES` | 4 of 5 are PLACEHOLDER | Time against the caption track |

Follow [SWAP-IN-REAL-VIDEO.md](SWAP-IN-REAL-VIDEO.md) for the chat timing work,
which is the only thing left on this page.

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

The joining link in your email needs `?c={{contact.id}}&t={{contact.session_time_iso}}`
on it. Without `?c=`, attendees arrive unidentified and no progress is
recorded. Without `&t=`, someone opening the link on a different device from
the one they registered on has no stored session there, and the room opens the
moment it loads instead of at their session time.

## Checkout

The low-ticket offer sold off the back of the masterclass: The Scoliosis
Solution, six weeks of access, $99 at the masterclass price against $199
normal and $905 stated value.

- Order summary carries the full value stack. The order bump (Scoliosis-Safe
  Yoga and Breathing, $24.30) is built but `BUMP_ENABLED` is `false`: the
  payment link carries one fixed amount and cannot charge both $99.00 and
  $123.30, so offering it would mean someone agreeing to the higher price and
  being charged the lower one for two products. Turn it on in GoHighLevel
  first, as a native bump or a second link, then flip the flag.
- `currentTotal()` is the single source of the amount to charge.

**The payment form is embedded, not linked to.** `PAYMENT_LINK` is loaded in an
iframe inside the order panel, so the card is typed on the processor's origin
and this page stays out of PCI scope, without the customer leaving the page.

**There is no contact step.** The hosted form collects first name, last name
and email itself, so a step above it asking for the same three meant typing
them twice. The frame's `src` carries what the funnel already knows
(`firstName`, `lastName`, `email`, plus `contact_id`), so an attendee who
registered arrives with it filled in and nothing to type but their card. A
cold visitor types them once, in the frame. The hosted form has no phone
field and ignores a `phone` parameter, so **this page no longer captures a
phone number** — registration still does.

Consequences worth knowing:

- **Frame heights are fixed**, measured from the real form (1010px, 1060px
  under 620px wide). A cross-origin frame cannot report its height, and the
  hosted page sends no resize message. Re-measure if that form gains a field.
- **The loading placeholder clears on a timer as well as on `load`.** That
  event takes upwards of ten seconds on the hosted page and does not always
  arrive at all. It also never takes pointer events, or it would have
  swallowed every click on the form underneath it.
- **A fallback link sits under the frame**, carrying the same prefill, for a
  browser or extension that blocks the embed.

### The guarantee length contradicts itself

The live page says **14 days** on its badge graphic and **30 days** in its FAQ,
on the same screen. This build routes every mention through one constant,
`GUARANTEE_DAYS`, currently 30. Confirm which is correct before launch: it is
a refund term, so the wrong number is a chargeback argument waiting to happen.

## Thank you page

Where the payment link redirects after a successful payment. Nothing links to
it; it is reached only by that redirect, which is set on the GoHighLevel side
once the domain is live. It carries `noindex`.

- **It breaks out of the checkout's frame, first thing.** The payment form is
  embedded, so the redirect after payment lands *inside* that iframe: without
  this the thank you page would render in a 1010px box, inside the checkout,
  with the old order summary still beside it.
- Three next steps: check your email, book the 1:1 Welcome Call, start week one.
- Greets them by first name when the funnel knows it, from storage or `?fn=`,
  and silently skips it when it does not. Written with `textContent`, because
  a name off the query string is not markup.
- The booking calendar is the GoHighLevel one, embedded from `CALENDAR_URL`
  at the top of the page's script (currently the `XRzk337QDvr7WL7ckbJX`
  widget). GHL's `form_embed.js` loads with it so the iframe sizes itself to
  its content; that resize works by messaging the iframe's `id`, so the `id`
  set alongside the `src` has to be the one GHL's own embed code hands out for
  that calendar. Change the calendar and both have to change together.
- Clear `CALENDAR_URL` and the page falls back to a **placeholder** block,
  deliberately drawn as an obvious gap rather than a pretty empty box, so an
  unfinished page is never mistaken for a finished one.

## Page weight and mobile

Every page carries `charset`, `viewport`, the brand favicon as an inline data
URI, `theme-color`, font preconnects and a shared mobile baseline block.

**`checkout.html` had no `viewport` meta at all**, so phones were rendering it
at desktop width and zooming out. That is fixed, and it is the first thing to
check on any page pasted in from elsewhere.

The favicon is inlined rather than linked because these pages get pasted into
GoHighLevel, where a relative path would break and an absolute one would couple
production to this repo. The master is only 24x24, which is all Squarespace
holds for the live site. A larger source is needed before an
`apple-touch-icon` is worth adding, or the home-screen icon will be soft.

Images were recompressed in place, which took **358 KB off two pages**:

| Page | Before | After |
|---|---|---|
| `registration.html` | 355 KB | 179 KB |
| `checkout.html` | 288 KB | 114 KB |

The logo strips quantise to 32 colours with an RMSE under 2, because they are
white marks on transparency and never needed truecolour. The photographs are
JPEG q72. Both were checked by eye, not just by the numbers. The images stay
inline as data URIs for the same GoHighLevel reason as the favicon.

The one remaining lever, if these ever need to be lighter, is moving the images
to the client's CDN and referencing them absolutely. That is a hosting decision
rather than a code one.

## Before launch

- [ ] Add the Meta pixel base code to all five pages. `fbq` is only called if already
      defined, so nothing breaks until then, and the Meta fields in the
      registration payload stay empty strings.
- [ ] Fire `Lead` and `Schedule` server-side from the Conversions API, passing
      the `event_id` from the registration payload. Neither fires from the
      browser here, on purpose.
- [ ] Set `LIVE_ROOM_URL` in `confirmation.html` if the waiting room is not at
      `live.html`, and repoint the footer nav on every page.
- [ ] Time `HOST_MESSAGES` against the caption track. See
      [SWAP-IN-REAL-VIDEO.md](SWAP-IN-REAL-VIDEO.md).
- [ ] Add `session_time_iso` as a GHL contact field and map it from the
      inbound webhook, or `&t=` on the joining link resolves to nothing.
- [ ] **Take the payment link out of TEST MODE.** The embedded form renders a
      yellow TEST MODE badge, so it charges nothing. This is the one blocker
      between the checkout looking finished and it taking money.
- [ ] The embedded form's country selector defaults to **Australia**; it
      should be the USA. No URL parameter changes it (`country`, `countryCode`,
      `billing_country` and `address[country]` were all tried), and the frame
      is cross-origin, so it is a GoHighLevel or Stripe account setting.
- [ ] Set `CALENDAR_URL` in `thank-you.html` to the GoHighLevel calendar.
- [ ] Point the payment link's post-payment redirect at `thank-you.html`.
- [ ] Reconcile brand tokens against the brand guidelines.
      See [../../docs/brand-tokens.md](../../docs/brand-tokens.md).

## Webhooks

Both are wired and verified. Payload shapes, field-by-field notes and the
debugging steps are in [WEBHOOK-PAYLOADS.md](WEBHOOK-PAYLOADS.md).

## Form fields

First name, last name, email, mobile, consent checkbox. Nothing else. Every
extra field costs completions, and anything about the person's curve is a
better conversation for the call than the opt-in.
