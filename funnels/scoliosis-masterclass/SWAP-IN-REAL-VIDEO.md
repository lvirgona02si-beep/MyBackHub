# Connecting the masterclass recording

The recording exists: **1920x1080, 37:48 (2268 seconds), 268MB**. It is not in
this repo. GitHub caps files at 100MB, and a 268MB direct-served MP4 is the
wrong delivery method for a 38 minute session anyway.

It goes on a video host. Vimeo or Tella, per the client's decision.

## 1. Connected

Hosted on Tella and wired:

```js
var EMBED_BASE = "https://www.tella.tv/video/vid_cmu716b8d00ww0agmcmnbhava/embed";
var PLAYER     = "tella";
```

`EMBED_BASE` carries no query string. The page appends its own per state:
muted or unmuted, and the start offset.

`EMBED_BASE` carries no query string on purpose. Tella's own snippet hardcodes
`muted=0&t=0`, which would defeat both the autoplay and the late join: browsers
refuse unmuted autoplay, and a fixed `t=0` restarts the session from the top for
everyone. The page supplies both per state instead.

## 2. The host must support seeking from the URL

This is the one thing to confirm before committing to a host. Vimeo takes the
offset as a media fragment (`#t=120s`), Tella as a query param (`&t=120`). Both
are already handled in `embedSrc()`.

Without URL seeking, late arrivals cannot be dropped into a session already in
progress, and the whole simulated-live premise collapses.

## 3. What is already handled

| Requirement | How |
|---|---|
| Autoplay on arrival | `autoplay=1&muted=1`. Browsers only permit autoplay while muted, and the click from the previous page does not carry as a gesture |
| Starts at the exact session time | The offset is computed from the stored session start and passed to the embed |
| Tap to unmute | The sound bar rebuilds the iframe unmuted **at the position reached**, not from the top |

The unmute rebuild works off `sessionOpenedAt`, the wall-clock time of video
position zero. Storing that rather than a counter means the rebuild lands
correctly however long someone takes to tap.

## 4. Progress tracking changed with the player

An embedded player exposes no playback position cross-origin, so progress is
now **visible time on the page**, not the video's own clock. The counter only
advances while the tab is in view.

That is less precise than reading `currentTime`, and it is the deliberate
trade for using a host that handles delivery. It still rules out the case that
matters: someone opening the room, walking away, and being tagged as having
watched to the end.

## 5. Still to do by hand

`HOST_MESSAGES` in `live.html`. Four of the six are PLACEHOLDER text:

| `at` | Content |
|---|---|
| `5` | Welcome. Real copy |
| `60` | PLACEHOLDER: why straight-spine therapy fails |
| `420` | PLACEHOLDER: the first secret |
| `900` | PLACEHOLDER: the rotation explanation |
| `1320` | Booking link. Real copy |
| `1680` | Booking link, second prompt. Real copy |

All six now fall inside the 2268 second runtime, so all six will fire. Watch
the recording with a stopwatch and move each `at` to the moment the presenter
reaches that point.

## 6. A note on the duration

The recording is 37:48. The customer-facing copy says "around 35 minutes" in
three places, and that is deliberate: it is a promise, not a measurement, and
the client chose to leave it. Every timing constant uses the real 2268 seconds,
so nothing in the tracking depends on the copy.

## 7. Then re-run the cases that need the real duration

- Arrive 2 minutes late, join at 2:00 with the chat backlog present
- Arrive past the 10 minute cap, fresh session from 0:00
- Background the tab 5 minutes, the counter barely moves
- Watch to 95 percent, `watched-complete` fires once

`DEBUG_PINGS` is on; the console reports every send and skip.
