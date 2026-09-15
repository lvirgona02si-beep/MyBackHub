# Webhook payloads

Two separate GHL inbound webhooks, one per event kind, so each workflow
receives only what it handles and cannot be triggered by the other.

Both verified: preflight returns 204 with permissive CORS, and a POST of the
shape below returns `200 {"status":"Success: test request received"}`.

| Event | Page | Hook id ends |
|---|---|---|
| Registration | `registration.html` | `ed1e3dd3-…` |
| Watch progress | `live.html` | `48c9302d-…` |

---

## Registration

Fires once, on successful form submit. The page does **not** redirect unless
this returns OK.

```json
{
  "type": "masterclass_registration",

  "first_name": "Jane",
  "last_name": "Doe",
  "email": "jane@example.com",
  "phone": "+15551234567",

  "session_time_iso": "2026-09-15T19:45:00.000Z",
  "session_time_local": "2:45 PM",
  "session_date_local": "Tue, 15 Sep 2026",
  "session_choice": "Starting shortly (2:45 PM)",
  "session_is_just_in_time": true,
  "timezone": "America/New_York",
  "timezone_offset_minutes": -240,

  "market": "United States",
  "funnel_type": "Masterclass",

  "event_id": "mbh-m1a2b3c4-x9y8z7w6",
  "fbclid": "",
  "fbp": "",
  "fbc": "",

  "page_url": "https://watch.mybackhub.com/sc-masterclass-registration",
  "referrer": "",
  "user_agent": "Mozilla/5.0 …"
}
```

### Session fields

`session_time_iso` is the machine value. **Merge the local pair into reminder
emails, not the ISO one.** Someone who booked "2:45 PM" needs reminding of
2:45 PM in their own zone. `timezone` is the IANA name from the browser, which
is also what to set on the GHL contact so every later send is correct.

`session_is_just_in_time` distinguishes someone starting in minutes from
someone booked hours out. They need different reminder cadences: the
just-in-time registrant needs nothing but the joining link, the later booking
needs a reminder nearer the time.

### Meta fields

`fbclid`, `fbp` and `fbc` are **empty strings until the pixel is on the page**
and a visitor has arrived through an ad click. They are always sent, so build
the CRM field mapping now and it starts populating the moment the pixel goes
live, with no change to the page.

`fbc` is synthesised from `fbclid` when the `_fbc` cookie has not been written
yet, using Meta's `fb.1.<timestamp>.<fbclid>` format. On a first pageview the
cookie often does not exist, and without this, match quality drops on exactly
the traffic you paid for.

`event_id` is the deduplication key. **Pass it to the Conversions API when it
fires `Lead`.** Only the CRM fires `Lead` today, but the moment a browser-side
event is added, the same id on both is what stops Ads Manager counting one lead
twice and doubling your reported cost per lead.

---

## Watch progress

Fires once per milestone: `watched-start`, `watched-25`, `watched-50`,
`watched-75`, `watched-complete`.

```json
{
  "type": "masterclass_progress",
  "tag": "watched-50",
  "percent": 50,
  "seconds_watched": 1050,
  "watched_at": "2026-09-15T18:35:00.000Z",
  "contact_id": "<from ?c= on the joining link>",
  "email": "<if known>",
  "first_name": "<if known>"
}
```

`tag` arrives ready-made, so the workflow applies it rather than branching on a
percentage.

Nothing is sent unless a `contact_id` or `email` is known: an empty identifier
reaching a Create/Update Contact action matches nothing and creates a blank
record. This is why opening `live.html` directly sends nothing. Add
`?c=TEST123`, or arrive through the confirmation page.

The joining link in your emails needs `?c={{contact.id}}`. It is carried onward
from confirmation to the room to the booking link, but it has to start there.

---

## Where contact_id comes from

Short version: **on a cold registration there is no contact id, and there
cannot be.** GHL mints it server-side, and the inbound webhook returns only
`{"status":"Success…"}`. The browser never learns it.

So identity travels two ways:

| Path | Identifier | Link carries `?c=` |
|---|---|---|
| Cold registration, same session | `email`, from stored `mbhAttendee` | No, and that is correct |
| Arrived from a GHL email with `?c={{contact.id}}` | `contact_id` | Yes, forwarded at every hop |

Progress pings accept **either**, which is why watch tracking still works on a
cold registration despite the join link having no `?c=`.

Each page now reads the URL first and falls back to the stored attendee record,
so once an id enters the funnel by any route it is carried to the end:
registration to confirmation to the room to the booking link. Registration also
sends `contact_id` in its payload when it has one, so the workflow updates that
contact rather than creating a duplicate.

Unsubstituted merge fields are rejected: a literal `{{contact.id}}` arriving
because the email editor failed to substitute is discarded rather than sent.

**If you want `?c=` on the join link for a cold registration**, the CRM has to
hand the id back. Either have the registration workflow respond with the
contact id and I will read it from the webhook response, or send people to the
room through the emailed joining link rather than the in-page button.

## Debugging

`DEBUG_REGISTRATION` in `registration.html` and `DEBUG_PINGS` in `live.html`
are both on. The console logs the full payload and the response status for
every send, and the reason for every skip. Turn both off once verified.

**Neither webhook can fire from the Artifact preview.** That sandbox blocks all
outbound requests to third-party hosts, silently. Test on a real host.
