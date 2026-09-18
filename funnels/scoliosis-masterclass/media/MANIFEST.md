# Media manifest

Local copies are a fallback. The published pages stream from the client's
filesafe CDN. Never point a production page at a file in this repo.

CDN base:
`https://assets.cdn.filesafe.space/wPy45DABxzg1naBrVAG8/media/transcoded_videos`

Per hash the CDN serves `<hash>.jpg` (poster, 1280x720) and
`cts-<hash>_360p|_480p|_720p|_1080p.mp4`. Both pages reference `_720p`.
Originals (1920x1080) are at
`https://storage.googleapis.com/msgsndr/wPy45DABxzg1naBrVAG8/media/<source id>.mp4`.

## Testimonials

Pulled from the pre-call FAQ page and matched to their on-page quotes by
duration, then verified frame by frame. Numbering is the order they appear
on the registration page.

| File | Quote | Length | CDN hash | Source id |
|---|---|---|---|---|
| `01-pain-free-within-first-week` | "…within the first week, I was having mostly pain free days." | 1:02 | `ec781cb117071fac` | `696f97594a6464701bc2edd9` |
| `02-real-exercises-for-my-back` | "…we're actually getting real exercises geared toward my back." | 2:09 | `08786fdd3fb3e36f` | `696febd97b1aed4fa7344a56` |
| `03-pain-since-my-thirties` | "Starting in my thirties, the pain was pretty bad…" | 2:14 | `f938a1a33dd5721e` | `696febae41fb811d77dd7752` |
| `04-pain-management-courses` | "The courses on pain management are amazing." | 0:22 | `d441db3b90cdad66` | `696f975915885ec81ef31baa` |
| `05-i-can-make-this-work` | "I can do this program. I can make it work." | 0:12 | `f3d15584946d51d4` | `696f97590732b472e0be6313` |
| `06-enjoying-the-classes` | "I enjoy all the classes that you are providing…" | 0:26 | `e09010705b84fc90` | `696f9759193b566800782085` |

Ordered longest and strongest first. The two-minute stories carry the most
weight, and `02` answers the objection the whole page is built around.

## Welcome

| File | Used on | Length | CDN hash | Source id |
|---|---|---|---|---|
| `dr-mike-welcome` | `confirmation.html` | 1:39 | `e6355af3cd1a8277` | `69936324097809d4bb5a6c80` |

## Local file specs

Videos are 854x480 CDN transcodes. Posters are the CDN's own 1280x720 frames.

## Checkout

| File | Used on | CDN / source |
|---|---|---|
| `checkout/member-testimonials.mp4` | `checkout.html` | `697a98dd3f6f1815358f14a9` |
| `checkout/member-testimonials.jpg` | poster frame | first frame of the above |
| `checkout/scoliosis-solution-logo.png` | order summary | `69d821de019dc508d3d5ccaa` |
| `checkout/product-stack.png` | not yet placed | `67d9dc429065218a9e28e6f3` |

The original testimonial video is 1920x1080 and **121MB**, and the live
checkout page serves it at full size. The copy here is 640x360 at 15MB. Upload
it and repoint `checkout.html` before running traffic: a 121MB download on the
page where people enter card details is a conversion problem, not a detail.
