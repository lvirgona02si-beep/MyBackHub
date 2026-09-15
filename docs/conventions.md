# Repo conventions

Read this before adding a funnel, a page, or a media file.

## Folder layout

```
funnels/<funnel-slug>/     one folder per funnel, self-contained
  <page>.html              one file per page, named by its job
  README.md                what the pages do, what to wire before launch
  media/                   media belonging only to this funnel
    MANIFEST.md            where each file came from, and its CDN URL
    <group>/               testimonials, welcome, hero, and so on
shared/                    anything used by more than one funnel
  brand/                   logos, banners, brand imagery
docs/                      cross-funnel documentation (this folder)
  just-in-time-webinar-logic.md   the JIT funnel build reference
  jit-implementation-notes.md     where this build diverges from it
```

The rule that decides where something goes: **if only one funnel uses it, it
lives inside that funnel's folder.** Promote a file to `shared/` the moment a
second funnel needs it, not in anticipation.

## Naming

- Folders and files are `lower-kebab-case`. No spaces, no capitals. A file
  called `Funnel Header banner.png` breaks shell commands and URLs.
- Pages are named by their job, not by the funnel: `registration.html`, not
  `sc-masterclass-registration.html`. The folder path already says which
  funnel it belongs to, so repeating it is noise.
- Media is named by **what it contains**, prefixed with the order it appears
  on the page: `02-real-exercises-for-my-back.mp4`. A file called `t4.mp4`
  forces whoever opens it next to play all six to find the one they want.
- A poster frame shares its video's basename with a `.jpg` extension, so the
  pair sorts together and the relationship is obvious.

## No `build/` folder

These pages are hand-written source with no build step. `build/` conventionally
means generated output that is safe to delete and regenerate, which is the
opposite of true here. The pages live in `funnels/`, and nothing in this repo
is generated.

## Hosted copies declare their own charset

Every page in `funnels/` starts with `<meta charset="utf-8">`. GitHub Pages
sends `charset=utf-8` in the response header, so the pages render correctly
there without it, which is exactly why the omission goes unnoticed. Paste the
same file into a builder that does not set the header, or open it over
`file://`, and every curly quote, arrow and degree sign turns to mojibake.

The Artifact copies do not carry it: that skeleton injects its own.

## Media and the CDN

The published pages stream video from the client's existing filesafe CDN, not
from this repo. Local copies exist as a fallback if those URLs ever move, and
are kept at 480p for that reason. Every media folder carries a `MANIFEST.md`
recording the source URL and identifiers for each file.

Keep it that way: never point a production page at a file in this repo.
