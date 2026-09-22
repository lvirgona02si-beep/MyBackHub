# MyBackHub

Funnel buildout for MyBackHub, a scoliosis pain programme.

## Layout

```
funnels/                    one folder per funnel, self-contained
  scoliosis-masterclass/
    registration.html       picks the session, posts to GHL
    confirmation.html       counts down, hands over to the room
    live.html               THE ROOM: player, chat, watch tracking
    checkout.html           the $99 offer sold off the back of it
    media/                  video and imagery for this funnel only
shared/                     assets used by more than one funnel
  brand/
docs/                       conventions, brand tokens, source funnels
```

If only one funnel uses a file, it lives inside that funnel's folder. Promote
it to `shared/` when a second funnel needs it, not before.

Read [docs/conventions.md](docs/conventions.md) before adding anything.

## Funnels

| Funnel | Pages | Status |
|---|---|---|
| [Scoliosis Masterclass](funnels/scoliosis-masterclass/) | Registration, confirmation, live room, checkout | Built. Both webhooks live. Room is on Wistia; chat cue timings are the last open item |

## Docs

| Doc | What it covers |
|---|---|
| [Conventions](docs/conventions.md) | Where files go and how they are named |
| [Brand tokens](docs/brand-tokens.md) | Palette, type, and house copy style |
| [Source funnels](docs/source-funnels.md) | Pages being replaced, and the structural reference |
| [JIT webinar logic](docs/just-in-time-webinar-logic.md) | How the just-in-time funnel works, end to end |
| [JIT implementation notes](docs/jit-implementation-notes.md) | Where this build diverges, and what is still needed |
| [Simulated-live spec](SIMULATED-LIVE-WEBINAR-SPEC.md) | The master build spec the live room is built to |

## Working on the pages

Every page is a single self-contained HTML file. No build step, no package
manager, no dependencies. Open one in a browser and it runs.

Colour and type live in one `:root` block at the top of each file. Change them
there rather than hunting through the stylesheet.

Two house rules worth knowing before you edit copy: coral is reserved for
call-to-action buttons and nothing else, and there are no em dashes in
customer-facing copy. Both are explained in
[docs/brand-tokens.md](docs/brand-tokens.md).
