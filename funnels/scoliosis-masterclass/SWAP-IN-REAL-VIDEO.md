# The masterclass recording

The recording exists: **1920x1080, 37:48 (2268 seconds), 268MB**. It is not in
this repo. GitHub caps files at 100MB, and a 268MB direct-served MP4 is the
wrong delivery method for a 38 minute session anyway. Wistia re-encodes to
H.264 and serves adaptive bitrate, so a phone is not pulling the master.

`.gitignore` blocks `*.mp4` and `*.mov` outright so a stray local copy cannot
break a push.

## 1. Connected

Hosted on Wistia and wired into `live.html`:

```js
var WISTIA_ID = "5i1tmdo2w2";
```

That is the only place the id is written. The page injects the per-media module
from it at reveal time.

Dashboard: <https://dylana057.wistia.com/s/a51zcyxo12t79ew>

## 2. Why the `<wistia-player>` web component, not `E-v1.js`

Its options are plain HTML attributes, present *before* the player initialises,
so they cannot arrive too late and be discarded the way an options object
pushed to `_wq` can. It also exposes `currentTime`, `duration`, `muted` and
`play()` the way a plain `<video>` does, which is what the watch tracking reads.

## 3. If an on-demand page is ever added, upload the file twice

A Wistia media carries its own player settings and **those beat the attributes
set on an embed for anything not named explicitly**. The live room needs no
controls and instant muted autoplay; a recording page needs a play button and a
scrub bar. One media cannot be both.

`live.html` names every control explicitly and sets it to `false`, so a change
in the dashboard cannot put a progress bar over a live session. An on-demand
page must name every control explicitly too, set to `true`, for the same reason
in reverse.

Two attribute spellings that were wrong for a long time in the reference build,
and both leak the illusion:

| Attribute | The wrong version |
|---|---|
| `copy-link-and-thumbnail` | `copy-link-and-thumbnail-enabled`, which is ignored, so "Copy Link and Thumbnail" stayed in the right-click menu, naming the media and linking the file |
| `wistia-logo` | Left out entirely. It sits outside the control bar, so switching every *control* off never touched it, and it is worst in fullscreen |

## 4. The runtime corrects itself

`MASTERCLASS_SECONDS` starts at 2268 and is overwritten with Wistia's own
`duration` the moment the player reports one. Every progress mark is a fraction
of it, so **the video can be re-cut without touching this page, the tag
thresholds, or anything in the CRM**.

The one thing that does not recompute is `LATE_JOIN_MAX_MS`, because it is
needed to decide which session to open before the player exists. It is
`min(10 minutes, runtime)`, so it only matters if the recording is ever cut
below ten minutes.

## 5. Still to do by hand: the chat timings

`HOST_MESSAGES` in `live.html`. Four of the five are PLACEHOLDER text:

| `at` | Content |
|---|---|
| `5` | Welcome. Real copy |
| `60` | PLACEHOLDER: why straight-spine therapy fails |
| `420` | PLACEHOLDER: the first secret |
| `900` | PLACEHOLDER: the rotation explanation |
| `2100` | The checkout link. Real, but the time is a guess |

**Anchor every cue to the caption track, never to the script.** Pull

```
https://fast.wistia.net/embed/captions/5i1tmdo2w2.vtt
```

and time each line against a phrase Dr. Mike actually says. Scaling from a
presenter script's section budgets ran the reference build **up to 3:15 late**
in the middle of its session, and the drift is not linear, so no single offset
fixes it.

Anything he **reads out loud** must be on screen *before* he reads it, 3 to 20
seconds ahead of its anchor. Getting that backwards has him answering a message
nobody sent yet, which is the single most obvious tell in the whole build. Mark
those lines `// LOCKED` as you write them, not afterwards.

Keep a companion table as the source of truth: line, old time, new time,
speaker, message, **anchor phrase in the transcript**. Edit there first, then
copy into the array. Re-cut the video and you re-pull the VTT; you do not
rescale.

## 6. Simulated attendees are wired but off

`SIM_ON` is `false` and `SIM_MESSAGES` is empty. The dual-array tick, the
separate fired maps and the host-only link gate are all in place, so writing
the cast and flipping the flag is all that is needed.

Two constraints on whoever writes that array:

- Attendee lines go through `addMessage` with **no host flag**, and only a host
  line can render a link. That is the safety property: nothing in the attendee
  feed can ever become a clickable URL.
- **This is a medical offer.** An invented attendee reporting that a treatment
  worked is a fabricated patient testimonial, which is an FTC problem before it
  is a taste problem. Arrivals, locations, logistics, questions and one honest
  sceptic are the material. Outcomes are not.

## 7. A note on the duration

The recording is 37:48. The customer-facing copy says "around 35 minutes" in
three places, and that is deliberate: it is a promise, not a measurement, and
the client chose to leave it. Every timing constant uses the real runtime, so
nothing in the tracking depends on the copy.

## 8. Cases to re-run once the chat is timed

1. `?in=30` — countdown hands over, player autoplays muted, sound bar works
2. Register on a phone, open the join link on a laptop: `&t=` must hold the time
3. Join 5 minutes late → seeked, chat backlogged, skipped marks **not** in the CRM
4. Close the tab mid-session, reopen → continues, does not restart
5. Background the tab 2 minutes → video pauses, `seconds_watched` does not advance
6. iOS Safari: player not collapsed, fullscreen overlay carries no Apple chrome,
   no long-press share sheet
7. Right-click / player menu: no "Copy Link and Thumbnail", no Wistia badge
8. Watch to 90 percent → `watched-complete` fires once

`DEBUG_PINGS` is on; the console reports every send and skip.
