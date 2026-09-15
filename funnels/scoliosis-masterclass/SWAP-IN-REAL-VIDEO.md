# When the real masterclass recording arrives

The live room currently plays Dr. Mike's welcome video as a stand-in. It is
**1:39 (99 seconds)**. The real session is expected to run about 35 minutes.

Everything below is in the `CONFIGURE ME` block at the top of the script in
`live.html`, unless stated otherwise.

## 1. The two constants that matter

```js
var MASTERCLASS_SRC     = "<url of the real recording>";
var MASTERCLASS_SECONDS = 35 * 60;   // the REAL runtime, in seconds
```

Set `MASTERCLASS_SECONDS` to the actual runtime, not the advertised one. If the
recording comes in at 41 minutes, put `41 * 60`.

## 2. What recalculates by itself

Nothing below needs touching. It is all derived from `MASTERCLASS_SECONDS`:

| Derived | How |
|---|---|
| Progress marks | `MARKS` are fractions (`0.25`, `0.50`, `0.75`, `0.95`), never hardcoded seconds |
| Late-join cap | `LATE_JOIN_MAX_MS = Math.min(10 * 60, MASTERCLASS_SECONDS) * 1000` |

**This is why late-joining looks broken right now.** With the 99 second
stand-in, the cap is 99 seconds, so arriving two minutes late gives a fresh
start rather than a two minute offset. It is working correctly. It cannot be
tested properly until the real duration is in.

## 3. What needs retiming by hand

`HOST_MESSAGES` in `live.html`. The `at` values are seconds of watched time,
and four of the six are placeholders waiting for the real recording:

| `at` | Currently | Fires on the stand-in? |
|---|---|---|
| `5` | Welcome message | Yes |
| `60` | PLACEHOLDER: why straight-spine therapy fails | Yes |
| `420` | PLACEHOLDER: the first secret | No |
| `900` | PLACEHOLDER: the rotation explanation | No |
| `1320` | Booking link | No |
| `1680` | Booking link, second prompt | No |

The timings are already pitched at a ~35 minute session, so they are roughly
right once the real video is in. What they are not is *matched to it*. Watch
the recording with a stopwatch and move each `at` to the moment the presenter
actually reaches that point, then replace the four PLACEHOLDER texts.

Keep the last two booking prompts in the final third. Earlier and they land
before anyone has a reason to book.

## 4. Also worth checking

- The `35 minutes` figure in the meta row of `live.html`, the stat strip on
  `registration.html`, and the FAQ answer "The masterclass will be around 35
  minutes". If the real runtime is materially different, these are three
  separate places that say so.
- `SESSION_MINUTES` in `confirmation.html`, which sets the calendar event's
  end time.
- `MASTERCLASS_SECONDS` in `registration.html`, which enforces the minimum gap
  between two sessions offered at once. A longer session means wider spacing,
  and `OFFSETS_MS` may need revisiting if the runtime goes past 60 minutes.

## 5. Then re-run the test matrix

From [the logic reference](../../docs/just-in-time-webinar-logic.md), the cases
that only become meaningful once the real duration is in:

- Arrive 2 minutes late, join at 2:00 with the chat backlog present
- Arrive past the cap, fresh session from 0:00
- Background the tab 5 minutes, `watchedSeconds` barely moves
- Watch to 95 percent, `watched-complete` fires once

`DEBUG_PINGS` is on in `live.html`; the console reports every send and skip.
Turn it off once verified.
