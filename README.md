# MyBackHub — Scoliosis Masterclass Funnel

Rebuild of the scoliosis masterclass registration and confirmation pages,
restructured for conversion. Both pages are self-contained HTML: no build
step, no dependencies, drop straight into GoHighLevel or any host.

## Pages

| File | Replaces |
|---|---|
| [build/sc-masterclass-registration.html](build/sc-masterclass-registration.html) | `watch.mybackhub.com/sc-masterclass-registration` |
| [build/sc-masterclass-confirmation.html](build/sc-masterclass-confirmation.html) | `watch.mybackhub.com/thank-you-sc-masterclass` |

Structure and copy style are modelled on a workshop funnel from a different
industry (Golden Key Property), adapted to scoliosis.

### What changed on the registration page

- Registration form moved above the fold, beside the headline. The live page
  renders a permanently broken `Getting sessions...` loading state where the
  session picker should be.
- Headline interrupts a decision in progress rather than describing a benefit.
- Six specific discovery bullets in place of three abstract ones.
- Session countdown, stat strip, and a "Worth Your Time If" qualification block.
- Six video testimonials pulled from the pre-call FAQ page, replacing the
  Elfsight text widget. See [build/testimonial-video-map.md](build/testimonial-video-map.md).
- Medical disclaimer added.
- Sticky mobile CTA bar.

### What changed on the confirmation page

- Live session countdown with a progress bar, flipping to an open/join state
  at zero. The current page never tells anyone when their session is.
- Google, Outlook and .ics calendar links generated from the session time.
- Checklist expanded to four items and rewritten for an exercise-based session.
- Post-session "Book A Call" CTA.
- Dr. Mike's welcome video carried over.

## Before launch

- [ ] Wire `#regForm` to the LeadConnector/GHL endpoint and repoint the success
      state at the real session room.
- [ ] Set the countdown to the true session cadence (currently rolls to the
      next `:00` or `:30`).
- [ ] Repoint placeholder links: waiting room, join, Book A Call, and the
      footer nav.
- [ ] Remove the build note from the footer of each page.
- [ ] Reconcile brand tokens against the brand guidelines doc. Colours and type
      are currently reverse-engineered from the live site CSS and are defined in
      one `:root` block at the top of each file.

## Assets

`Assets/` holds the header banner and local copies of the testimonial videos
(480p) with their poster frames. The published HTML streams video from the
existing filesafe CDN rather than these files; they are here as a fallback if
those URLs ever move.
