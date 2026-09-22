# Simulated-Live Webinar — Master Build Spec

How the Golden Key Property UK workshop funnel works, written so it can be
rebuilt for a different niche without reading the original source. Everything
niche-specific is marked `«REPLACE»`.

The system is **static HTML on GitHub Pages**. No server, no framework, no
build step. Every page is a single file with inline CSS and one inline
`<script>`. The only third parties are Wistia (video), GoHighLevel (CRM, via
inbound webhooks) and Meta (pixel).

Nothing on the page ever reads from the CRM — that would need an API key, and
an API key cannot live in a file anyone can view-source. All state is either in
the browser's own storage or passed on the URL.

---

## 1. The shape of it

```
/«funnel»/                    registration — picks the session, posts to GHL
/«funnel»/confirmation.html   holding page — counts down, hands over
/«funnel»/live.html           THE ROOM — player, chat, tracking
/«random-path»/               byte-identical copy of the room, for SMS links
/«funnel»/watch.html          on-demand, honest about being a recording
/«funnel»/recap.html          short cut, for people who left early
/«funnel»/booking.html        call calendar
/«funnel»/video.js            every media ID, in one place
/«funnel»/workbook.pdf        fill-in PDF referenced during the session
```

Five pages share one scheduling contract and one set of storage keys. Get those
right and the rest is presentation.

**Sessions run just in time on the quarter hour.** The registration page picks
the next slot, the confirmation page counts down to it, the room opens the
player at that moment and autoplays. Someone arriving late is dropped into the
session already in progress, not handed a fresh countdown.

### The mirror page (`/ua3d2c/`)

An exact copy of `live.html` at a meaningless path. Australian carriers
silently drop SMS whose links read as marketing infrastructure — a path like
`/uk-workshop/live` is enough to trigger it. **Every SMS** uses the mirror;
every email uses the real path. Keep them byte-identical (`diff` them in CI or
before each release) and put a note on the CRM custom value explaining why it
duplicates, because this is what gets "tidied" back into one link.

---

## 2. Scheduling contract

Shared by registration, confirmation and the room. Copy these constants
verbatim into all three.

```js
const SLOT_MS      = 15 * 60 * 1000;   // sessions on the quarter hour
const MIN_LEAD_MS  =  3 * 60 * 1000;   // minimum time to fill the form
const SESSION_KEY  = 'gkWorkshopStart';
const CHOICE_KEY   = 'gkWorkshopChoice';
const ENTERED_KEY  = 'gkWorkshopEntered';   // room only
```

### Storage: always mirror to both

`sessionStorage` **and** `localStorage`, every write, read session-first:

```js
function readStart() {
  let v = 0;
  try { v = parseInt(sessionStorage.getItem(SESSION_KEY) || '0', 10) || 0; } catch (e) {}
  if (!v) { try { v = parseInt(localStorage.getItem(SESSION_KEY) || '0', 10) || 0; } catch (e) {} }
  return v;
}
function rememberStart(t) {
  try { sessionStorage.setItem(SESSION_KEY, String(t)); } catch (e) {}
  try { localStorage.setItem(SESSION_KEY, String(t)); } catch (e) {}
}
```

Someone clicking the confirmation email arrives in a **fresh browser session**
where `sessionStorage` is already gone. Without the localStorage mirror they
look like a brand new visitor and get handed a second countdown. Every access
is wrapped in `try/catch` — private mode and blocked site data both throw.

### Picking the slot (registration page only)

Rolling forward to the next slot is correct **here and only here**: nobody has
registered yet, so there is no session they are late for.

```js
function workshopStartAt() {
  const now = Date.now();
  let target = readStart();
  if (!target || target <= now || target - now > SLOT_MS + MIN_LEAD_MS) {
    target = Math.ceil(now / SLOT_MS) * SLOT_MS;       // next :00/:15/:30/:45
    if (target - now < MIN_LEAD_MS) target += SLOT_MS; // too close to register
    rememberStart(target);
  }
  return target;
}
```

### The later-session picker

A dropdown: the soonest session preselected, plus three fixed offsets.

```js
const OFFSETS_MS    = [60 * 60000, 3 * 3600000, 6 * 3600000];
const START_HOUR    = 9;              // nothing offered before 9am
const CUTOFF_HOUR   = 21;             // after 9pm, later slots roll to tomorrow
const ROLLOVER_HOURS = [9, 12, 17];   // tomorrow: 9am / noon / 5pm
```

Rules that matter and are not obvious:

- **No offset shorter than the session itself.** A 30-minute workshop cannot
  start 15 minutes after the previous one; offering both reads as a recording
  rather than a schedule.
- **Later sessions round up to :00 or :30, never :15 or :45.** A published
  timetable does not run at quarter past. Quarter-hour starts are the tell that
  the times were generated from the visitor's own clock. The *soonest* session
  keeps the finer 15-minute rounding because it is labelled "starting shortly"
  rather than as a scheduled time.
- **An offset landing outside 9am–9pm rolls to a fixed slot** rather than being
  dropped. Without this, someone browsing at 8pm sees every later offset fall
  past the cutoff and gets a near-empty dropdown that looks broken.
- **Never offer the same slot twice** — seed a `seen` map with the soonest slot.

The option labels: `Today at 4:15 pm · starts in 07:42` (rewritten every second
by the countdown tick) and `Today at 6:00 pm (in 1 hour)` / `Tomorrow at 9:00 am`.

### On submit

```js
sessionStorage.setItem('gkAttendee', JSON.stringify({ first_name, email }));
sessionStorage.setItem(CHOICE_KEY, JSON.stringify(chosenSession));
localStorage .setItem(CHOICE_KEY, JSON.stringify(chosenSession));
if (chosenSession.jit) rememberStart(chosenSession.at);
```

`chosenSession` is `{ jit: bool, at: epochMs, label: string }`.

**Note the asymmetry, it has bitten before:** `SESSION_KEY` is only written for
a *just-in-time* registration, because the countdown on the opt-in button
rewrites it every second to the next quarter hour. A booked 4pm lives in
`CHOICE_KEY` and nowhere else. Downstream pages that read only `SESSION_KEY`
will open the room against a stale rolling slot instead of the booking.

---

## 3. The room: opening logic

The room resolves a start time from three sources, in this order.

```js
function bookedStart(now, params) {
  // 1. ?t= on the link — the only source that survives a change of device.
  const raw = params.get('t');
  if (raw && !/[{}]/.test(decodeURIComponent(raw))) {
    const v = /^\d+$/.test(raw) ? parseInt(raw, 10) : Date.parse(raw);
    if (isFinite(v) && v > 0 && Math.abs(v - now) < 24 * 3600000) return v;
  }
  // 2. CHOICE_KEY, for a session booked for later.
  let choice = null;
  try { choice = JSON.parse(sessionStorage.getItem(CHOICE_KEY) || 'null'); } catch (e) {}
  if (!choice) { try { choice = JSON.parse(localStorage.getItem(CHOICE_KEY) || 'null'); } catch (e) {} }
  if (choice && choice.jit === false && choice.at) {
    const v = parseInt(choice.at, 10);
    if (isFinite(v) && v > 0) return v;
  }
  return 0;   // 3. caller falls back to readStart()
}
```

`?t=` accepts ISO or epoch ms, and is **ignored if more than a day from now or
if it contains braces** — an unresolved `{{contact.session_time_iso}}` merge
field arriving literally must not be acted on. Same guard on `?c=`. The failure
is then silent and safe: the page falls back to storage.

### Resolution, in order

```js
const now = Date.now();
const params = new URLSearchParams(window.location.search);
const previewIn = parseInt(params.get('in') || '', 10);
let target;

if (isFinite(previewIn) && previewIn >= 0 && previewIn <= 3600) {
  target = now + previewIn * 1000;              // ?in=30 — demo hook, never persisted
} else {
  target = bookedStart(now, params);
  if (target) {
    rememberStart(target);                       // survives a refresh without ?t=
  } else {
    target = readStart();
    if (target - now > SLOT_MS + MIN_LEAD_MS) target = 0;   // implausible, discard
  }
  const returning = !!target && readEntered() === target &&
                    (now - target) < WORKSHOP_SECONDS * 1000;
  if (!target || (!returning && now - target > LATE_JOIN_MAX_MS)) {
    target = now; rememberStart(target);
  }
}
sessionTarget = target;
if (target <= now) { reveal((now - target) / 1000); return; }   // late join, seeked
```

Four rules encoded there, each learned the hard way:

1. **A start time in the past means they are late, not that they need a fresh
   one.** Arriving at 7:02 for a 7:00 session drops them two minutes in, running.
2. **`LATE_JOIN_MAX_MS = min(10 min, runtime)`.** Past that they have missed too
   much to follow it, so a new session starts from the top.
3. **`ENTERED_KEY` distinguishes returning from late.** Closing the tab and
   reopening it used to restart the workshop at zero — and nothing says
   "recording" louder than that. A browser that has already been inside this
   session carries on wherever the clock is now, however long the tab was shut.
   That is what a live call does.
4. **Nothing stored at all means the browser lost the booking**, not that no
   booking was made. Open the session now rather than restarting a wait they
   have already sat through.
5. **A booked time skips the plausibility guard.** That guard exists to throw
   away a stale rolling slot; a session legitimately booked six hours out would
   trip it.

`?in=<seconds>` is the demo hook: starts a countdown of that length from load,
wins over everything, resets on reload, and is deliberately **not persisted** —
a preview must not leave a fake booking in the browser of someone who later
arrives for a real session. It can only ever *delay* the room opening. There is
no parameter that reveals a session early or skips part of one.

### `reveal(offsetSec)`

The room is on screen from the moment they arrive — the wait happens *inside*
it, not in front of a closed door. `reveal()` clears the waiting state:

- `rememberEntered(sessionTarget)`
- hide the countdown stage; remove the player placeholder and the chat idle text
- drop `.is-idle` on the chat body, hide the chat footer, unhide the composer
- add `.is-live` to `.player-wrap` (max-width 1040px → 1280px, so the panels
  **visibly grow** at the moment the session opens)
- `buildPlayer(offsetSec)`, unhide the sound bar and the fullscreen button
- `startEngagementTracking(offsetSec)`
- badge → red pulsing dot + "Live Now"; headline → "Under Way" if `offsetSec > 30`,
  else "Ready"

---

## 4. Wistia setup — the part that sells it

Player chrome is what gives away that a live session is a file. Everything here
exists to remove it.

### Two uploads of the same video, not one

```js
window.GK_WORKSHOP = {
  wistiaId:      '«live room media ID»',   // controls OFF in the dashboard
  watchWistiaId: '«on-demand media ID»',   // ordinary Wistia defaults
  recapWistiaId: '«short cut media ID»',
  poster: '/images/workshop-poster.jpg',   // from the recording's own title card
  seconds: 1886                            // real runtime; corrected from Wistia on load
};
```

A Wistia media carries its own player settings and **those beat the attributes
set on an embed**. The live room needs no controls and instant muted autoplay;
the recording page needs a play button and a scrub bar. One media cannot be
both. Upload the file twice. Changing either in the dashboard then only affects
its own page.

### Why the `<wistia-player>` web component, not `E-v1.js`

Its options are plain HTML attributes, present *before* the player initialises,
so they cannot arrive too late and be discarded the way an options object
pushed to `_wq` can. It also exposes `currentTime`, `duration`, `muted` and
`play()` the way a plain `<video>` does — which is what the watch tracking reads.

### Embedding

```html
<script src="https://fast.wistia.com/player.js" async></script>
<script src="/«funnel»/video.js"></script>
```

Then, at reveal time, inject the per-media module from the resolved ID so
`video.js` stays the one place any ID is written:

```js
const mod = document.createElement('script');
mod.src = 'https://fast.wistia.com/embed/' + MEDIA + '.js';
mod.async = true; mod.type = 'module';
document.head.appendChild(mod);
```

### Every control off, by name

```js
video = document.createElement('wistia-player');
video.setAttribute('media-id', MEDIA);
video.setAttribute('aspect', '1.7777777777777777');
[['autoplay','true'], ['muted','true'], ['silent-autoplay','allow'],
 ['controls-visible-on-load','false'], ['playbar','false'],
 ['play-button','false'], ['small-play-button','false'],
 ['volume-control','false'], ['fullscreen-button','false'],
 ['settings-control','false'], ['quality-control','false'],
 ['playback-rate-control','false'], ['play-pause-notifier','false'],
 ['copy-link-and-thumbnail','false'],   // NOT ...-enabled; that spelling is ignored
 ['resumable','false'],
 ['wistia-logo','false'],               // sits outside the control bar, worst in fullscreen
 ['do-not-track','true'],
 ['end-video-behavior','default']
].forEach(function (a) { video.setAttribute(a[0], a[1]); });
```

Two that were wrong for a while and both leak the illusion:
`copy-link-and-thumbnail` (mis-spelled with `-enabled`, so "Copy Link and
Thumbnail" stayed in the right-click menu, naming the media and linking the
file) and `wistia-logo` (outside the control bar, so switching every *control*
off never touched it).

The on-demand page names **every control explicitly too**, set to `true`. The
dashboard can turn controls off for a whole media, so leaving them implicit
would let a settings change there put a progress bar over a live session.

### Readiness poll

There is no reliable ready event. Poll, and give up rather than spin:

```js
let tries = 0;
const ready = setInterval(function () {
  const d = video.duration;
  if (d && isFinite(d) && d > 0) {
    clearInterval(ready);
    WORKSHOP_SECONDS = d;                                  // Wistia's runtime beats the constant
    if (seekTo > 0) { try { video.currentTime = Math.min(seekTo, d - 1); } catch (e) {} }
    try { video.play(); } catch (e) {}
  } else if (++tries > 60) { clearInterval(ready); }
}, 500);
```

Runtime correcting itself from Wistia is what lets you re-cut the video without
touching `video.js`, the tag thresholds, or anything in the CRM.

### Autoplay, sound, and the phone

Muted autoplay is the one form every browser permits without a gesture, so the
session starts silent behind a pulsing **"Tap for sound"** bar. The click that
got them here happened on the previous page and does not carry.

On that tap: **play first, unmute second.** Unmuting before the video is
running turns a request the browser would have allowed into one it can refuse
outright. And the bar only goes **once playback is actually under way** —
`play()` returns a promise, a rejection never reaches a `try/catch`, and the
shield covers the whole video surface, so removing the bar on a failed tap left
the player with no control that could reach it and no way back but a reload.

```js
bar.addEventListener('click', function () {
  if (!video) { bar.remove(); return; }
  let p; try { p = video.play(); } catch (e) { return; }
  if (p && typeof p.then === 'function') {
    p.then(function () { try { video.muted = false; } catch (e) {} bar.remove(); })
     .catch(function () {});          // refused: bar stays, next tap tries again
  } else { try { video.muted = false; } catch (e) {} bar.remove(); }
});
```

### Three layers of pause defence

1. **A transparent shield** (`position:absolute; inset:0; z-index:2`) over the
   player, under the sound button. Wistia still toggles play/pause on a click to
   the video surface even with every control off.
2. **`-webkit-touch-callout: none` and `user-select: none`** on the player,
   shield and fill. A long press on iOS otherwise raises the system sheet —
   *Save Video, Copy, Share* — which says plainly that this is a file on a server.
3. **A `pause` listener that restarts it.** A phone stops playback on its own:
   an incoming call, another app taking the audio, Low Power Mode. A live
   session sitting quietly paused is the worst failure here.

```js
video.addEventListener('pause', function () {
  if (document.visibilityState !== 'visible') return;   // ours, on hide
  if (video.ended) return;
  const d = video.duration;
  if (d && isFinite(d) && video.currentTime >= d - 1.5) return;   // run-out
  setTimeout(function () {
    if (video && video.paused && document.visibilityState === 'visible') resumePlayback();
  }, 250);
});
video.addEventListener('playing', clearResume);
```

`resumePlayback()` calls `play()`, and on a promise rejection appends a **"Tap
to resume"** button. This matters on a phone, where a notification banner, the
app switcher, the share sheet and the control centre all fire
`visibilitychange`, the handler pauses — and then iOS refuses `play()` on an
unmuted video without a fresh gesture. It failed silently for a while because
the rejection is a promise, which no `try/catch` sees.

### Fullscreen — never ask the video element

```js
function setFs(on) {
  grid.classList.toggle('is-fs', on);
  document.body.classList.toggle('is-fs-lock', on);
  fsLabel.textContent = on ? 'Exit full screen' : 'Full screen';
  fs.setAttribute('aria-pressed', on ? 'true' : 'false');
}
```

On an iPhone, video fullscreen is the *only* fullscreen available, and it draws
Apple's own player over the top — scrub bar, skip buttons, time remaining on the
file — in the middle of a session billed as live. So the room drives a **CSS
class overlay** (`position:fixed; inset:0`) on the whole grid, and layers a real
`grid.requestFullscreen()` on top of it only where the browser supports one.
Both look identical. Listen for `fullscreenchange` so Esc doesn't leave the
overlay up over a page that is no longer fullscreen.

### Layout traps worth copying verbatim

- **The 16:9 box comes from `padding-top: 56.25%` on a `::before`, not from
  `aspect-ratio`.** Every real child is absolutely positioned, so the moment
  `reveal()` removes the idle placeholder there is nothing in flow left to
  measure. Chrome resolves `aspect-ratio` anyway; **Safari collapses the player
  to a sliver.** A pseudo-element is always in flow.
- **The chat is `position:absolute` pinned to the second column**, not a grid
  item in it. In flow its height came from its own content — every message
  released so far — and a grid row sizes to its tallest item, so a long session
  stretched the row and the player with it.
- **`.chat-body { min-height: 0 }`.** A flex item defaults to `min-height:auto`
  and refuses to shrink below its content, so the scrollbar never appears and
  the feed just gets taller.
- **`.live-grid > * { min-width: 0 }`.** Grid children default to
  `min-width:auto`. Wistia writes an iframe carrying its own width attribute,
  and on a phone that one element sets the floor for the whole row and shoves
  the page sideways.
- **`.player-fill { position:absolute; inset:0 }` wrapping the player**, sized
  by insets rather than `height:100%`. A percentage height inside a flex
  container whose own height comes only from `aspect-ratio` resolves to zero on
  iOS Safari. The wrapper has to be ours because Wistia writes inline styles
  onto the player element, and inline beats a stylesheet rule.
- **`.msg-from`, `.msg-text { overflow-wrap: anywhere }`.** A pasted URL sets a
  minimum width the column must honour.
- Under 900px the grid goes one column and the chat returns to static flow with
  `min-height:220px; max-height:380px` so it scrolls rather than running away.

---

## 5. The chat

A **one-way feed on video time**, driven from the same 1s tick as the tracking,
so a line lands against the right part of the talk even for someone who paused
or joined late. `at` is seconds of *watched video*, never wall clock.

```js
const HOST_MESSAGES = [
  { at: 0,    from: '«Host»', text: "Hi guys welcome in don't forget to grab the workbook on your way in" },
  { at: 1757, from: '«Host»', text: 'https://…/booking.html', link: { href: 'https://…/booking.html' } }
];

const firedMessages = {};
function releaseDueMessages(seconds) {
  for (let i = 0; i < HOST_MESSAGES.length; i++) {
    const m = HOST_MESSAGES[i];
    if (!firedMessages[i] && seconds >= m.at) {
      firedMessages[i] = true;
      addMessage(m.from, m.text, { host: true, link: m.link });
    }
  }
}
```

### Rendering rules

`addMessage(from, text, { mine, host, link, note })`:

- **`textContent` always**, never `innerHTML`. A typed question can never inject
  markup.
- **`link` is honoured only when `host` is set.** The viewer's own messages go
  through the same function, so gating on `opts.host` is what stops typed text
  ever rendering a link.
- Host links get `target="_blank" rel="noopener noreferrer"`. Following the
  booking link in place walked people out of the workshop, and the way back is a
  reload that drops them into a session well past where they left it. `rel` goes
  with `target` or the booking page gets a handle on this one via `window.opener`.
- `chatBody.scrollTop = chatBody.scrollHeight` after each append.
- Host lines get a gold left border and heavier text, so the feed always reads
  as one host addressing the room.

### The composer

The recording asks people four times to put an answer in the chat, so there has
to be somewhere to put one — **a prompt with no input is worse than no prompt.**
What goes in stays on that screen: it is not posted anywhere.

```js
form.addEventListener('submit', function (e) {
  e.preventDefault();
  const text = input.value.trim();
  if (!text) return;
  addMessage('You', text, { mine: true });
  input.value = '';
});
```

### Simulated attendees

The feed runs two arrays through the same tick. `HOST_MESSAGES` are the host's
own lines; `SIM_MESSAGES` is the room. Separate `fired` maps, one loop each, and
a flag so the room can be switched off whole:

```js
const SIM_ON = true;
const SIM_MESSAGES = SIM_ON ? [
  { at: 3,   from: 'Marcus Reid',   text: 'Hello' },
  { at: 374, from: 'Tom Whitfield', text: 'Yield not covering loan' },   // LOCKED
  { at: 632, from: 'Nathan Okonjo', text: "Isn't this the thing that had the regulator all over it a few years back?" },  // SEEDED
  …
] : [];

for (let j = 0; j < SIM_MESSAGES.length; j++) {
  const s = SIM_MESSAGES[j];
  if (!firedSim[j] && seconds >= s.at) { firedSim[j] = true; addMessage(s.from, s.text, {}); }
}
```

**They go through `addMessage` with no `host` flag**, which is the safety
property that matters: only host lines can render a link, so nothing in the
room's feed can ever become a clickable URL.

For this niche the feed was written from a reference cast
(`working-files/webinar-chat-sim/cast-of-30.md`) — 30 named attendees with a
region, a capital position, a strategy history and a **chat voice**, which is
how they type. **You are writing the new industry's chat by hand**, so the cast
file is the thing worth porting, not the messages. The rules that make it read
as a room rather than as one writer:

- **Not everyone posts.** 19 of 30 post, 11 never do. A 100%-participation chat
  is the single biggest tell. Lurkers still earn their place — two of them book
  at the end, which is true to life.
- **Consistent voice per person.** One types in caps when annoyed, one
  abbreviates everything, one writes full sentences with proper punctuation,
  one drops words because they type fast. Hold each one steady all session.
- **A spread of capital tiers**, including people who cannot transact. The
  under-threshold ones asking "can I start smaller?" is realistic and hands the
  host a natural qualification moment.
- **One informed sceptic**, who remembers whatever bad press the sector has. A
  room with nobody raising the obvious objection is not credible — and the
  host's script already pre-empts it, so the sceptic is the reason that line
  exists.
- **One or two professionals** either side of the subject, asking the questions
  that make the session look like it survived scrutiny.
- **Attendees answer each other**, not just the host. That is what a real chat
  does and it is mostly absent from faked ones.

Two conventions to carry into the new array, both as trailing comments:

- **`// LOCKED`** — a line the host **reads out loud**. It must be on screen
  before he reads it. See the timing rule below; this is the constraint that
  actually binds.
- **`// SEEDED`** — an objection planted a few seconds before the host answers
  it, so his answer lands as a response rather than as a topic he raised
  himself. The risk questions, the pricing question and the "do I actually own
  it" question are all seeded here.

Density follows the session's structure rather than being even: a burst of
arrivals in the first minute, a burst at each workbook activity, a burst at the
booking link, and a closing wave of ~25 one-word lines in the last 40 seconds.
Flat density reads as a script.

### Timing the feed — the one rule that matters

**Anchor every cue to the caption track, never to the script.**

Pull `https://fast.wistia.net/embed/captions/«MEDIA_ID».vtt` and time each line
against a phrase the host actually says. The first pass was scaled from the
presenter script's section budgets and ran **up to 3:15 late** in the middle of
the session — and the drift is not linear, so no single offset fixes it.

Lines the host **reads out loud** are `[LOCKED]`: they must be on screen
*before* the cue. Getting this backwards has him answering a message nobody
sent yet, which is the single most obvious tell in the whole build. Every
locked cue should sit 3–20s ahead of its anchor.

Keep a companion markdown table (`webinar-chat-timings-corrected.md`) as the
source of truth — line, old time, new time, speaker, message, **anchor phrase
in the transcript** — and edit there first, then copy into the array. Re-cut the
video and you re-pull the VTT; you do not rescale.

---

## 6. Engagement tracking

One 1s interval drives both the chat and the milestone tags.

```js
let WORKSHOP_SECONDS = (window.GK_WORKSHOP && window.GK_WORKSHOP.seconds) || 1886;
const MARKS = [
  { at: 0.25, ev: 'Workshop25',       tag: 'watched-25',       pct: 25 },
  { at: 0.50, ev: 'Workshop50',       tag: 'watched-50',       pct: 50 },
  { at: 0.75, ev: 'Workshop75',       tag: 'watched-75',       pct: 75 },
  { at: 0.90, ev: 'WorkshopComplete', tag: 'watched-complete', pct: 100 }
];
```

**`watched-complete` fires at 90%, not 100%.** People close the tab in the last
seconds of an outro they have effectively finished, and holding out for the full
length loses most of the audience that actually watched it. This tag gates the
guide and the calculator, so erring generous is the right way to be wrong.

### The tick

```js
function engagementTick() {
  const now = Date.now();
  if (document.visibilityState === 'visible') {
    let t = null;
    if (video) { try { t = video.currentTime; } catch (e) { t = null; } }
    if (t !== null && isFinite(t) && t > 0) {
      watchedSeconds = Math.max(watchedSeconds, t);   // monotonic: a mark can never un-fire
    } else if (!video && lastTick !== null) {
      watchedSeconds += (now - lastTick) / 1000;      // wall clock ONLY with no player at all
    }
    lastTick = now;
  } else {
    lastTick = null;                                   // hidden tab accrues nothing
  }
  releaseDueMessages(watchedSeconds);
  for (…) if (!firedMarks[m.ev] && watchedSeconds >= WORKSHOP_SECONDS * m.at) { … }
}
```

Progress is **real playback**, not time on the page. A stalled connection no
longer walks someone up to `watched-complete` without them seeing a frame.

The `!video` guard on the wall-clock branch is load-bearing: a player that
*exists* but reports nothing is the normal state on a phone where autoplay was
refused and the video sits waiting for a tap. Counting seconds there walked
those viewers to `watched-complete` with no frame played — and that tag is the
one signal the whole follow-up branches on.

### Visibility

```js
document.addEventListener('visibilitychange', function () {
  if (document.visibilityState === 'visible') {
    lastTick = Date.now();
    if (video && video.paused) resumePlayback();
  } else {
    lastTick = null;
    if (video) { try { video.pause(); } catch (e) {} }
  }
});
```

The room **pauses the video on hide and resumes on show**: a backgrounded tab
accrues nothing *and* nobody misses content.

### Late joiners

```js
function startEngagementTracking(offsetSec) {
  watchedSeconds = offsetSec;
  for (let i = 0; i < MARKS.length; i++) {
    if (WORKSHOP_SECONDS * MARKS[i].at <= offsetSec) firedMarks[MARKS[i].ev] = true;
  }
  fireMeta('WorkshopStart');
  pingProgress('watched-start', 0);
  releaseDueMessages(offsetSec);   // chat backlog, as on a real call joined late
  …
}
```

Marks the offset skipped past are **recorded as fired but never sent**. They
reached that point of the video; they did not watch it. Tagging them for it
poisons the one signal used to tell an engaged lead from a browser.

### The on-demand page is different

`watch.html` has a scrubber, so it **accumulates played seconds in small steps**
rather than reading the playhead — otherwise dragging to the end earns a
`watched-complete`, and that tag gates the reward content.

```js
const pos = video ? video.currentTime : null;
if (pos !== null && isFinite(pos)) {
  if (lastPos !== null && pos > lastPos && pos - lastPos < 2) watchedSeconds += pos - lastPos;
  lastPos = pos;
}
```

Same tags, different Meta event names (`Recording25` …), so the nurture sequence
segments someone who watched on demand exactly as if they had watched live.

---

## 7. CRM integration (GoHighLevel)

### One inbound webhook per payload type

So a workflow can only ever receive what it handles. `type` is still sent on
every payload as a sanity check and to keep the data self-describing, but it is
not load-bearing.

| Payload | Sent by |
|---|---|
| `workshop_registration` | registration form submit |
| `workshop_progress` | each milestone, from the room and the on-demand page |

### Registration payload

```
type, first_name, last_name, email, phone,
country, country_code,              ← from the phone field; stops GHL defaulting to UK
session_choice,                     ← "Today at 4:15 pm"
session_time_iso,                   ← REQUIRED, see below
session_is_just_in_time,            ← true = watching now, false = booked later
marketing_consent, consent_text,
source, market, funnel_type,
event_id, fbclid, fbp, fbc, user_agent
```

### Progress payload

```js
function pingProgress(tag, pct) {
  const attendee = getAttendee();
  if (!attendee.email && !attendee.contact_id) return;   // never create a blank record
  const payload = { type: 'workshop_progress', tag, percent: pct,
                    seconds_watched: Math.round(watchedSeconds),
                    watched_at: new Date().toISOString() };
  if (attendee.contact_id) payload.contact_id = attendee.contact_id;
  if (attendee.email)      payload.email      = attendee.email;
  if (attendee.first_name) payload.first_name = attendee.first_name;
  fetch(WEBHOOK, { method: 'POST', headers: {'Content-Type':'application/json'},
                   body: JSON.stringify(payload) }).catch(function () {});
}
```

**Only send identifiers you actually have** — an empty email reaching a
Create/Update Contact action matches nothing and creates a blank record.
`.catch(…)` swallowing everything is deliberate: tracking never blocks the player.

The tag is sent **ready-made**, so the workflow is one action — add
`{{inboundWebhookRequest.tag}}` — with no branching on a percentage.

### Identifying the viewer

```js
function getAttendee() {
  let a = {};
  try { a = JSON.parse(sessionStorage.getItem('gkAttendee') || '{}') || {}; } catch (e) {}
  let cid = new URLSearchParams(window.location.search).get('c');
  if (cid && /[{}]/.test(decodeURIComponent(cid))) cid = null;   // unresolved merge field
  if (cid) { a.contact_id = cid; try { sessionStorage.setItem('gkAttendee', JSON.stringify(a)); } catch (e) {} }
  return a;
}
```

Read **lazily**, not at load, so attribution does not depend on script order.

### The room link

```
https://«domain»/«funnel»/live?c={{contact.id}}&t={{contact.session_time_iso}}
https://«domain»/«mirror»/?c={{contact.id}}&t={{contact.session_time_iso}}     ← SMS only
```

Store both as CRM custom values so they cannot drift.

**`session_time_iso` must exist as a contact field *and* be mapped from the
inbound webhook**, or the merge resolves to nothing, `&t=` does nothing, and the
room silently reverts to opening the moment it loads. Send yourself a test and
click it — the address bar has to show a real timestamp, not `%7B%7B…`.

### GHL build gotchas

- **Dynamic tags:** the Add Tag field has a hidden Standard/Dynamic toggle
  behind the three-dot menu. Switch to Dynamic before a merge field is accepted.
- **Number fields** reject typed merge syntax; insert through the merge picker.

### Follow-up branching

Contacts accumulate tags, so branch on the **furthest** one present:

| Furthest tag | Branch |
|---|---|
| `watched-complete` (90%+) | **A — Full.** Guide, then calculator, then the sequence |
| `watched-25/-50/-75` | **B — Partial.** No guide. One re-engagement email with a 5-min condensed cut + booking link, then rejoins A |
| none / `watched-start` only | **C — Missed.** No guide. One "you missed it" email linking the next session, then rejoins A |

The reward content is the reward for finishing; B and C never get it, so
anything later in the sequence referring back to "the guide" has to be reworded
on those paths. **These three groups behave nothing alike — sending them the
same sequence is the usual reason webinar follow-up underperforms.**

---

## 8. Meta pixel

| Event | Fired by | Where |
|---|---|---|
| `PageView` | page | all pages |
| `ViewContent` | page | room load — the "actually attended" signal |
| `WorkshopStart` | page | player opened |
| `Workshop25/50/75/Complete` | page | watched-time marks, with `seconds_watched` |
| `RecordingStart/25/50/75/Complete` | page | on-demand equivalents |
| `CompleteRegistration` | **CRM, server-side** | **optimise the campaign on this** |
| `Schedule` | **CRM, server-side** | value on booked call. Track, do not optimise |

`Lead` is deliberately **not** fired browser-side — it goes out server-side via
the CRM's Conversions API, to avoid double-counted Leads in Ads Manager.

`Purchase` is deliberately not sent at all: a 60–90 day sales cycle is well
outside Meta's 7-day attribution window, so the event arrives unattributed. The
expected value sits on `Schedule`.

For dedup and match quality, the form generates and sends `event_id`, `fbc`,
`fbp`:

```js
const eventId = 'lead_' + Date.now() + '_' + Math.random().toString(36).slice(2);
const fbclid = new URLSearchParams(location.search).get('fbclid') || '';
let fbc = getCookie('_fbc');
if (!fbc && fbclid) fbc = 'fb.1.' + Date.now() + '.' + fbclid;
const fbp = getCookie('_fbp');
```

`event_id` is generated unconditionally, so if it is empty on a contact the
**mapping** is broken, not the page.

---

## 9. Other pieces worth carrying over

**Phone field, loaded async with a hard timeout.** A blocking `<script src>` for
`intl-tel-input` stalls the whole page — and so the form — on one third party
for as long as the browser takes to give up. That took out a previous funnel's
opt-in rate for a day during a CDN outage.

```js
const PHONE_LIB_TIMEOUT_MS = 2500;
(function loadPhoneLib() {
  const s = document.createElement('script');
  s.src = 'https://cdn.jsdelivr.net/npm/intl-tel-input@23.8.0/build/js/intlTelInputWithUtils.min.js';
  s.async = true; s.onload = initPhoneField; s.onerror = initPhoneField;
  document.head.appendChild(s);
  setTimeout(initPhoneField, PHONE_LIB_TIMEOUT_MS);
})();
```

`initPhoneField` is idempotent and falls back to a plain field with a
UK-normalising `phoneValue()`. The form is usable within 2.5s regardless, and
quietly upgrades if a late response arrives.

**Honeypot:** `if (document.getElementById('hpField').value) return;`

**Swap only the button's label, never its innerHTML.** The countdown holds a
direct node reference to `#cdBtn` inside the button; replacing the innerHTML
leaves the timer frozen and orphaned.

**Calendar links on the confirmation page** for a booked-later session — Google,
Outlook and a `data:text/calendar` ICS built inline. No library.

**`SESSION_ON_HOLD`** — a boolean at the top of both the confirmation page and
the room. While true, the room redirects back to the holding page and the
holding page swaps the countdown for a "here's the written guide instead" panel.
Registration, the pixel and the webhook are deliberately untouched, so opt-in
rate still measures cleanly on a day with no session running.

**`gkOriginPage`** in sessionStorage, so the shared booking page can route its
"back to homepage" link to whichever funnel sent the visitor.

**`<meta name="robots" content="noindex, nofollow">`** on the room and the mirror.

**`prefers-reduced-motion`** kills the pulsing live dot and the sound-bar pulse.
A flashing element is a genuine accessibility problem.

**Video hosting, not in the repo.** The master here is 3840×2160 HEVC at 1.02 GB.
GitHub rejects any file over 100 MB, and HEVC does not play in Firefox and only
conditionally in Chrome — so serving it raw was never an option either. Wistia
re-encodes to H.264 and serves adaptive bitrate, so a phone is not pulling 4K.
`.gitignore` blocks `*.mp4` and `*.mov` outright so a stray local copy cannot
break a push.

---

## 10. Porting checklist

**Replace:**

- [ ] `video.js` — all three media IDs, `poster`, `seconds`
- [ ] `HOST_MESSAGES` — re-timed against the **new** caption track
- [ ] `SIM_MESSAGES` — written by hand for the new industry. Build the cast
      first, then the lines, then anchor them to the VTT. Mark `LOCKED` and
      `SEEDED` as you go, not afterwards
- [ ] Meta pixel ID (every page)
- [ ] Both webhook URLs (registration, progress)
- [ ] Calendar ID, `EVENT_TITLE`, `EVENT_DETAIL`
- [ ] Storage key prefix `gk…` → your own, so two funnels on one domain cannot collide
- [ ] Brand tokens in `:root`, fonts, wordmark SVG, favicons
- [ ] `source`, `market`, `funnel_type` on the registration payload
- [ ] Booking link inside the final `HOST_MESSAGES` entry
- [ ] Workbook / lead-magnet paths
- [ ] A fresh meaningless mirror path (a reused one is already burned)
- [ ] `START_HOUR` / `CUTOFF_HOUR` / `ROLLOVER_HOURS` for the audience's timezone
- [ ] `OFFSETS_MS` — never shorter than the new runtime

**Keep unchanged unless you have a reason:**

- [ ] The whole Wistia attribute list, spellings included
- [ ] Shield, touch-callout suppression, pause listener, `resumePlayback`
- [ ] The `::before` padding-ratio box and the absolute chat
- [ ] Session resolution order, `ENTERED_KEY`, `LATE_JOIN_MAX_MS`
- [ ] The dual session/local storage mirror
- [ ] The `{}` guard on `?c=` and `?t=`
- [ ] `Math.max` monotonic progress and the `!video` wall-clock guard
- [ ] The 90% complete threshold
- [ ] Late-join marks fired-but-not-sent
- [ ] `textContent` everywhere, `link` gated on `host`
- [ ] Two Wistia uploads, not one

**Test before launch:**

1. `?in=30` — countdown hands over, player autoplays muted, sound bar works
2. Register on a phone, open the room link on a laptop — `&t=` must hold the time
3. Join 5 min late → seeked, chat backlogged, skipped marks **not** in the CRM
4. Close the tab mid-session, reopen → continues, does not restart
5. Background the tab 2 min → video pauses, `seconds_watched` does not advance
6. iOS Safari: player not collapsed, fullscreen overlay has no Apple chrome, no
   long-press share sheet
7. Right-click / player menu: no "Copy Link and Thumbnail", no Wistia badge
8. On-demand page: drag the scrubber to the end → **no** `watched-complete`
9. Real SMS to an Australian number via the mirror link — does it arrive?
10. `diff` the room against the mirror — must be byte-identical
