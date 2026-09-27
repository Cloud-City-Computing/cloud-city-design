# cloud-city-design

The shared design primitives of the Cloud City product suite: one stylesheet of
literal CSS custom properties, the two typefaces the suite ships, the design
gates that check a consumer's code against them, and a manifest that lets a
consumer vendor an exact, verifiable copy.

Version 0.2.0. Licensed under the Apache License 2.0 (see `LICENSE` and
`NOTICE`). The Apache License 2.0 grants no rights to the Cloud City, Cloud
Command or Cloud Codex names or marks (Apache License 2.0, section 6).

## What is in the package

| File | What it is |
|---|---|
| `core.css` | 61 primitives, declared on a single `:root` rule |
| `fonts.css` | `@font-face` rules for Inter and Poppins, with relative `./fonts/` URLs |
| `fonts/*.woff2` | Inter (variable, latin and latin-ext) and Poppins 500 and 600 (latin and latin-ext): six files |
| `fonts/OFL-Inter.txt`, `fonts/OFL-Poppins.txt` | The SIL Open Font License texts for those files |
| `brand.json` | The brand, accent and semantic colours as `#rrggbb`, generated from `core.css` |
| `gates/contrast.mjs` | WCAG contrast maths and a theme parser for stylesheets of custom properties |
| `gates/token-discipline.mjs` | Three source rules: literal colours, accent fill shades in text positions, suppressed focus outlines |
| `gates/dangling.mjs` | The dangling-var rule: a `var()` reference nothing declares |
| `gates/brand.mjs` | The generator for `brand.json` |
| `gates/*.test.mjs` | The package's own `node:test` suites, five files |
| `gates/fixtures/fx-*` | The fixture corpus the suites run over, with its golden findings in `fx-expected.json` |
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

## The gates

`gates/` holds plain ES modules with no dependencies: they import `node:`
built-ins and each other, and nothing else, so a consumer runs them with no
install. Each rule returns findings as plain data, and a consumer comparing two
runs compares `(rule, file, line)`; the `message` is for people.

### `token-discipline.mjs`

- `scanRawColorLiterals(source, file)`: rule `token-literal-color`. A 3-, 4-, 6-
  or 8-digit hex, or any colour function (`rgb`, `rgba`, `hsl`, `hsla`, `oklch`,
  `oklab`, `lch`, `lab`, `hwb`, `color`). A hex right after `href=` is a
  fragment and is not reported.
- `scanAccentRampPositions(source, file, { extraNames })`: rule
  `token-accent-position`. An accent fill shade in a `color`, `fill`, `stroke`,
  `outline`, `outline-color`, `box-shadow` or `border-color` position. A fill
  shade as a background is the correct use and is not reported. The banned
  names are `DEFAULT_ACCENT_FILL_NAMES` (`--brand-blue` and `--accent-100` to
  `--accent-400`) plus `extraNames`, where a consumer lists its own fill
  aliases; an entry that is not a custom-property name throws.
- `scanSuppressedOutlines(source, file)`: rule `token-outline-suppressed`.
  `outline: none` and every spelling that does the same (`0`, `0px`, `0em`,
  `0rem`, `0%`, `outline-style: none`, `outline-width: 0`), quoted or not.
- `maskComments(source)`: comment content blanked, with every UTF-16 offset and
  newline kept, so line numbers hold. Strings are tracked, so a `//` inside a
  quoted URL is not a comment.
- `collectScanTargets(base, roots, extensions = DEFAULT_EXTENSIONS)`: every file
  under `roots` whose name ends in one of `extensions`
  (`['.ts', '.tsx', '.css', '.html']` by default). A root that is a file is
  taken as is.

A comment reading `invariant-exempt: <reason>`, in either comment syntax,
exempts its own line and the line below it.

### `dangling.mjs`

- `scanDanglingVars(source, file, declared)`: rule `dangling-var`. Every
  `var(--name)` without a fallback whose name is not in `declared`.
- `declaredNames(source)` and `collectDeclaredNames(base, declarationRoots)`:
  the names a stylesheet declares. Only `.css` files contribute declarations.
- `inspectDangling({ base, declarationRoots, referrerRoots, extensions })`: the
  gate over a tree, with what it read (`referrerFiles`, `referenceSites`,
  `referencedNames`, `declared`), so a test can prove it was not looking at
  nothing. `runDangling(options)` returns the findings only. There are no
  default roots, and `extensions` reaches the referrer walk, so a code base in
  plain JavaScript passes a list with `.js` and `.jsx`.

### `contrast.mjs`

- `parseColor(value)`: an OKLCH value or a 3- or 6-digit hex, to sRGB. An
  out-of-gamut OKLCH value is clamped per channel, which is the conservative
  direction for a contrast check. Any other notation is `undefined`.
- `contrastRatio(foreground, background)`: WCAG 2.x, 1 to 21, with a
  translucent foreground flattened first (`compositeOver`).
- `parseThemeTokens(css, { isBase, themes })`: `{ light, ...themes }`. `light`
  is every block `isBase` accepts, merged in source order (by default, any
  selector mentioning `:root`, tested first). Each theme is `light` with its
  own blocks laid over it; a block joins the first theme whose pattern matches.
  The default `themes` is `{ dark: /\[data-theme=['"]dark['"]\]/ }`.
  `loadThemeTokens(path, options)` reads a file.
- `resolveTokenValue(tokens, name)` and `resolveToken(tokens, name)`: follow
  `var()` chains inside one theme, to the literal text or to its colour. A
  cycle is `undefined`; a `var()` with a fallback is returned as text.

### Documented behaviours

These are kept on purpose, so the gates stay equal to the implementation they
were ported from. A consumer that needs otherwise passes its own options.

- Quotes are tracked in every file type, so an apostrophe in HTML text opens a
  "string" that runs to the next apostrophe, and a comment inside it is not
  masked.
- An unquoted `url(//host/path)` in a stylesheet reads as a `//` comment.
- In the block-comment form, `/* invariant-exempt: */` exempts, because the
  closing `*/` counts as the reason. A `//` marker with nothing after the colon
  exempts nothing.
- A `:root[data-theme="dark"]` block mentions `:root`, so the default parse puts
  it in `light`. A `:root` block nested in an at-rule merges into `light` too.
- `parseColor` returns `undefined` for 4- and 8-digit hex, although the literal
  rule reports them.
- An id selector spelled in hex digits (`#fade`) is reported as a literal.
- Longhands such as `border-top-color` are not accent positions.

### Running the suites

```sh
node --test
```

from the package root, with no `package.json` and no install. The suites were
run on Node 22.22.2 and Node 20.20.2; no older release is claimed. The
fixtures are named so that `node --test` never collects one: every basename
starts with `fx-`, none matches a test-file pattern, and no directory is named
`test`.

## brand.json

`brand.json` carries the 20 brand, accent and semantic colours as `#rrggbb`,
for a consumer that can only take hex (an identity provider's label policy, an
email template). It has no neutrals, because `core.css` declares none:
surfaces, text and borders are a consumer's bindings.

The conversion is the contrast gate's own: OKLCH to sRGB with each channel
clamped, rounded to 8 bits, so a hex here is the colour every contrast figure
was measured on. Five colours fall outside sRGB and are listed under
`clamped`; they round-trip to their OKLCH source within 0.02 deltaE OK, and
every other colour within 0.002. `source.sha256` is the hash of the `core.css`
it was generated from. Regenerate it with `node gates/brand.mjs`; its suite
fails when it is stale.

## Vendoring contract

A consumer vendors the package rather than installing it from a registry.

1. Pick a commit on `main`. Only a merged commit is vendored.
2. Check that the checkout is clean and that every file `MANIFEST.json` lists
   has the SHA-256 it records.
3. Copy exactly the files `MANIFEST.json` lists into the vendor directory, and
   remove anything else there. The listed set for 0.2.0 is 37 files:
   `core.css`, `fonts.css`, `brand.json`, the six `.woff2` files, the two OFL
   texts, `LICENSE`, `NOTICE`, the four gate modules, their five suites, and
   the fifteen files under `gates/fixtures/`.
4. Write the consumer's own `MANIFEST.json` beside them: this repository's
   `MANIFEST.json` plus an `upstream` block naming where the copy came from.

```json
{
  "name": "cloud-city-design",
  "version": "0.2.0",
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

### In a consumer's tooling

Two things in a vendored copy trip tools that scan a whole tree.

**gitleaks.** A default gitleaks run reports two `generic-api-key` findings in
`MANIFEST.json`: the rule reads the word "token" in the
`gates/token-discipline.mjs` and `gates/token-discipline.test.mjs` keys and
takes the SHA-256 beside each for a credential. They are file digests, and
`gates/manifest.test.mjs` checks every one against its file. Allowlist them
narrowly: the one rule, the one file and a secret that is exactly a lowercase
SHA-256 digest, all three required, so the same digest in any other file is
still reported. For a copy vendored at `vendor/cloud-city-design/`:

```toml
[[allowlists]]
description = "SHA-256 digests in the vendored cloud-city-design MANIFEST.json"
targetRules = ["generic-api-key"]
condition = "AND"
paths = ['''(^|/)vendor/cloud-city-design/MANIFEST\.json$''']
regexTarget = "secret"
regexes = ['''^[0-9a-f]{64}$''']
```

Measured with gitleaks 8.30.1, in `git` and `dir` mode alike: two findings
without the entry, none with it, and one when the same digest sits in another
file.

**Linters, type checkers and bundlers.** The files under `gates/fixtures/` are
scanner input, not code: the suites read them as text and never import them,
and `fx-component.jsx` is not valid JSX on purpose (`href={#cafe}` puts a
literal colour where an expression belongs, and the literal-colour rule must
still find it). A consumer whose linter, type checker or bundler globs its
vendor directory excludes `gates/fixtures/`.

## Licence headers

Every stylesheet and module opens with the header: `core.css` and `fonts.css`
in their opening comment, and each gate module and suite as its first two
lines.

```js
// SPDX-License-Identifier: Apache-2.0
// Copyright 2026 Cloud City Computing, LLC
```

JSON has no comments, so `brand.json` states the same two facts as its first
two keys, `license` and `copyright`. `LICENSE` and `NOTICE` carry no header,
being the licence texts, and neither do the fonts and their OFL texts, which
are not Apache-2.0. The fixtures under `gates/fixtures/` carry none either:
their lines and comments are what the suites test, so a header would move
every golden line. They are Apache-2.0 under `LICENSE`, like the rest of the
package outside the fonts.

## Fonts and their licence

The font files in `fonts/` are not covered by the Apache License 2.0. Inter and
Poppins are licensed under the SIL Open Font License, Version 1.1, and each
licence text sits beside the files it covers (`fonts/OFL-Inter.txt`,
`fonts/OFL-Poppins.txt`). Redistribute them with those texts. The OFL governs
the font software only: everything in the package outside `fonts/*.woff2` and
the two OFL texts is Apache-2.0.

Inter is a variable font: one file per subset carries every weight from 100 to
900, declared as `font-weight: 100 900`. Poppins ships as two static weights,
500 and 600.

## Line endings

`.gitattributes` stores text files with LF endings and marks `.woff2` files as
binary, so a checkout on any platform hashes to the values in `MANIFEST.json`.
