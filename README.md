# cloud-city-design

The shared design primitives of the Cloud City product suite: one stylesheet of
literal CSS custom properties, the two typefaces the suite ships, and a
manifest that lets a consumer vendor an exact, verifiable copy.

Version 0.1.0. Licensed under the Apache License 2.0 (see `LICENSE` and
`NOTICE`). The Apache License 2.0 grants no rights to the Cloud City, Cloud
Command or Cloud Codex names or marks (Apache License 2.0, section 6).

## What is in the package

| File | What it is |
|---|---|
| `core.css` | 61 primitives, declared on a single `:root` rule |
| `fonts.css` | `@font-face` rules for Inter and Poppins, with relative `./fonts/` URLs |
| `fonts/*.woff2` | Inter (variable, latin and latin-ext) and Poppins 500 and 600 (latin and latin-ext): six files |
| `fonts/OFL-Inter.txt`, `fonts/OFL-Poppins.txt` | The SIL Open Font License texts for those files |
| `LICENSE`, `NOTICE` | Apache License 2.0 and the attribution notice |
| `MANIFEST.json` | The SHA-256 of every distributable file |

`README.md` and `.gitattributes` are repository files: they are not part of the
distributable set and a consumer does not copy them. `MANIFEST.json` is not
listed in itself and is not copied byte for byte either: the consumer writes
its own copy, this file extended with an `upstream` block (step 4 of the
vendoring contract below).

## The primitives

| Family | Names |
|---|---|
| Brand | `--brand-blue` |
| Accent ramp | `--accent-100` to `--accent-700` (seven steps) |
| Semantic ramp | `--{success,warning,danger,violet}-{deep,bright,pale}` |
| Faces | `--font-brand` (Poppins), `--font-ui` (Inter), `--font-mono` (system stack, no bundled face) |
| Type scale | `--text-xs`, `-sm`, `-base`, `-md`, `-lg`, `-xl`, `-2xl`; `--leading-tight`, `--leading-normal` |
| Spacing | `--space-1` to `--space-6`, and `--space-8` (4px base) |
| Radii | `--radius-sm`, `-md`, `-lg`, `-pill` |
| Motion | `--dur-fast`, `--dur-base`, `--ease-out` |
| Identity tints | `--avatar-1` to `--avatar-8` |
| Stacking | `--z-base`, `--z-sticky`, `--z-scrim`, `--z-modal`, `--z-skip`, `--z-menu`, `--z-toast` |

The semantic ramp has three steps per meaning because one lightness cannot
serve every field: `-deep` is for light surfaces, `-bright` for dark ones, and
`-pale` for text on mid-dark surfaces between `#1e1e1e` and `#444444`, where
each pale step clears 4.6:1. The danger and violet pale steps give up some
chroma to stay inside sRGB.

## The no-alias rule

`core.css` holds literals only. It has no theme selector, no media query and
no custom-property reference, and it never names a meaning such as "the
background" or "the danger colour". Those are the consumer's bindings:

```css
/* The consumer's own stylesheet, imported after core.css */
:root { --app-error: var(--danger-deep); }
.app-dark { --app-error: var(--danger-bright); }
```

A consumer never redeclares a name that `core.css` defines. A value that
should change is changed here, released, and vendored again.

## Using it

Import `core.css` before any other stylesheet. It opens with a `/*!` licence
comment, and CSS minifiers generally keep such a comment only when it is the
first thing in the output, so importing it first is what keeps the notice in a
built bundle.

Import `fonts.css` through a bundler. Its URLs are relative (`./fonts/...`), so
the bundler resolves and fingerprints the font files and nothing is requested
from a third-party host. A consumer that must serve the fonts from a fixed path
(for example to preload them by a stable name) keeps its own copy of the
`@font-face` rules with the same descriptors and its own URLs, and copies the
font files byte for byte.

## Vendoring contract

A consumer vendors the package rather than installing it from a registry.

1. Pick a commit on `main`. Only a merged commit is vendored.
2. Check that the checkout is clean and that every file `MANIFEST.json` lists
   has the SHA-256 it records.
3. Copy exactly the files `MANIFEST.json` lists into the vendor directory, and
   remove anything else there. The listed set for 0.1.0 is 12 files:
   `core.css`, `fonts.css`, the six `.woff2` files, the two OFL texts,
   `LICENSE` and `NOTICE`.
4. Write the consumer's own `MANIFEST.json` beside them: this repository's
   `MANIFEST.json` plus an `upstream` block naming where the copy came from.

```json
{
  "name": "cloud-city-design",
  "version": "0.1.0",
  "license": "Apache-2.0",
  "files": { "core.css": "<sha256>", "...": "..." },
  "upstream": {
    "repository": "https://github.com/Cloud-City-Computing/cloud-city-design",
    "commit": "<40-character commit SHA>"
  }
}
```

The consumer's `MANIFEST.json` is the one vendored file that is not a byte copy
of this repository, because a commit cannot contain its own hash. Every other
vendored file is byte-identical to the file at the named commit, and its
`files` map is equal to this repository's `MANIFEST.json` at that commit.

A change to a vendored file is proposed here first, then vendored again. It is
never made in the consumer's copy.

## Fonts and their licence

The font files in `fonts/` are not covered by the Apache License 2.0. Inter and
Poppins are licensed under the SIL Open Font License, Version 1.1, and each
licence text sits beside the files it covers (`fonts/OFL-Inter.txt`,
`fonts/OFL-Poppins.txt`). Redistribute them with those texts. The OFL governs
the font software only; `core.css` and `fonts.css` are Apache-2.0.

Inter is a variable font: one file per subset carries every weight from 100 to
900, declared as `font-weight: 100 900`. Poppins ships as two static weights,
500 and 600.

## Line endings

`.gitattributes` stores text files with LF endings and marks `.woff2` files as
binary, so a checkout on any platform hashes to the values in `MANIFEST.json`.
