# Scoliosis Masterclass funnel

Two pages, hand-written, no build step. Drop straight into GoHighLevel or any
static host.

| File | Replaces |
|---|---|
| `registration.html` | `watch.mybackhub.com/sc-masterclass-registration` |
| `confirmation.html` | `watch.mybackhub.com/thank-you-sc-masterclass` |

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

## Before launch

- [ ] Wire `#regForm` to the LeadConnector/GHL endpoint and repoint the success
      state at the real session room. It is front-end only right now.
- [ ] Set both countdowns to the true session cadence. They currently roll to
      the next `:00` or `:30`.
- [ ] Repoint placeholder links: waiting room, join, Book A Call, footer nav.
- [ ] Remove the build note from the footer of each page.
- [ ] Reconcile brand tokens against the brand guidelines.
      See [../../docs/brand-tokens.md](../../docs/brand-tokens.md).

## Form fields

First name, last name, email, mobile, consent checkbox. Nothing else. Every
extra field costs completions, and anything about the person's curve is a
better conversation for the call than the opt-in.
