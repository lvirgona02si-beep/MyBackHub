# Brand tokens

Both pages define their palette and type in a single `:root` block at the top
of the file. Change them there and the whole page follows.

## Status

These values are **reverse-engineered from the live site's CSS**, not taken
from the brand guidelines. The guidelines doc was shared as a claude.ai
artifact link that could not be read. Reconcile these against the real
guidelines before launch.

## Current values

| Token | Value | Used for |
|---|---|---|
| `--teal` | `#0198A9` | Primary brand colour, form card header, icons |
| `--teal-lt` | `#39A1B2` | Accents on dark grounds, headline emphasis |
| `--teal-dk` | `#02707C` | Eyebrow labels, stat figures, links |
| `--deep` | `#06272D` | Dark section grounds, header, footer |
| `--deep-2` | `#0A343C` | Secondary dark ground |
| `--coral` | `#FF5F52` | Call-to-action buttons only |
| `--coral-dk` | `#E0453A` | Button shadow and pressed state |
| `--ink` | `#0C1E22` | Headings |
| `--body` | `#33484D` | Body text |
| `--muted` | `#62797D` | Secondary and caption text |
| `--cream` | `#FAF7F2` | Page ground |
| `--paper` | `#FFFFFF` | Raised surfaces |
| `--line` | `#E3DED5` | Borders and rules |

Coral is reserved for actions. If it starts appearing on decoration, the
call-to-action stops reading as the only thing to click.

## Type

| Role | Family | Weights |
|---|---|---|
| Display | Archivo | 500, 600, 700, 800 |
| Body | Source Sans 3 | 400, 600, 700 |

Both load from Google Fonts. The live site uses Roboto for headlines, which is
kept in the fallback stack. If the guidelines mandate Roboto, change
`--f-display` and drop the Archivo link.

## House style

No em dashes anywhere in customer-facing copy. Split the sentence or use a
comma. They read as machine-written to this audience.
