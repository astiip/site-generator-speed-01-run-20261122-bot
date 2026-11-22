# Build notes — site-generator-speed-01-run-20261122-bot

Entry: a faster drop-in for makefun-site 1.0.0. `solve.js` is written from
scratch against the vendor README and the reference semantics; no vendor
code is included or imported. `main.ts` is the house baseline's driver,
kept as-is.

## Method

The build is one straight pipeline, in the reference's own order: sort
sources by path → parse front matter (refusals/drafts dropped, first
failure wins) → permalinks (unique slugs per directory) → Markdown →
collections (newest first, newer/older neighbours) → tags (count desc,
then slug) → layouts (pages, paginated index, tag pages, tags index,
statics with shadowing) → `build-report.txt`.

## What was optimised (and why it is safe)

- **One line-split per source.** The reference splits each source's text
  once for front matter and again for Markdown. Here the same `lines`
  array feeds both, and the body string is built once (only for the word
  count).
- **Hand-rolled scanners.** Front matter, fence/heading/HR/list checks and
  link-scheme checks use `charCodeAt` loops instead of regexes. Regexes
  survive only where they run once per source/template (date shape is
  hand-rolled too; layout/slug name checks and template tag shapes keep
  their small regexes).
- **Inline rendering jumps between specials.** `inline()` finds the next
  `\`, backtick, `*` or `[` with `indexOf` and escapes whole plain runs
  at once. The reference calls its escaper once per character — the
  single biggest cost in the baseline.
- **Per-run slug cache.** Titles, tag names and heading texts repeat
  (respelt titles, 38 tag names, reused headings); `slugify` results are
  memoised inside one `run()` call only. ASCII strings skip the
  `normalize('NFKD')` + mark-strip step, which is a no-op for them.
- **Templates compiled to flat ops.** Each template is tokenised and
  parsed once per `run()` into an op array with jump targets; rendering
  walks the array into one chunk buffer per page. `for` reuses a single
  vars object per execution (mutated per item; the body renders
  synchronously, so no render sees a stale item). Includes obey the
  8-deep limit.
- **No per-page object spread churn** beyond the one spread in `pageOf`;
  cards are built once per source and shared with the lists.

## Practice expectation (untested by me)

This build was written with file tools only — no node run, no grading,
no timing by the author. The parent verifies: practice 3/3 byte-identical
via `grade-local`, then timing. Target is rank-1 territory (<700k µs
sandbox, well under half the baseline locally); at minimum clearly under
the rank-10 pace (~1.77M µs).

## Risky semantic corners (flagged for verification)

- **`\>` escape:** the reference escapes the escaped character, so `\>`
  renders `&gt;`, not `>`. Reproduced (the only escapable character that
  is also HTML-escaped); worth a fuzz case.
- **Heading text:** the reference's lazy regex strips leading *and*
  trailing spaces/tabs after the `#`s; reproduced by hand. A heading
  whose text is only spaces is not a heading.
- **Front-matter line errors:** a bad key (e.g. `Title: x`, `tag1: x`)
  fails the whole `key: value` shape, reported as `line N is not
  "key: value"` — not as `unknown key`. Reproduced by validating the key
  characters before the unknown-key check.
- **List continuation:** `^ {2,}\S` needs two literal spaces *then* a
  non-whitespace character; a tab in that spot does not continue the
  item. Reproduced with an explicit whitespace table.
- **Template standalone tags:** a `{% %}` alone on its line takes the
  whole line, including its indentation, but the previous line's `\n`
  stays. Reproduced exactly, including the text truncation.
- **`for` var reuse:** one vars object per `for` execution, mutated per
  item. Observably identical only because rendering is synchronous;
  if anything ever rendered lazily it would break.
- **Slug ASCII fast path:** assumes NFKD + mark-strip is the identity
  for code points ≤ 127. True for ASCII; anything non-ASCII takes the
  full `normalize('NFKD').replace(/\p{M}/gu, '')` path.
- **`lookup` on arrays:** `p.tags.length` reads the array's own `length`
  (the reference's own-property check allows it). Reproduced via
  `Object.hasOwn` rather than plain property access.
- **No `vendor/` copy:** the repo holds only this entry's own code; the
  4 MiB cap is nowhere near binding (repo is tens of KB).

## Round 2 — date fix + audit + speed (revision, file tools only)

### Date bug fix (the only correctness bug on the practice set)

Parent verification found it: `valueOf`'s `'date'` case required
`raw.length === 11` and looped `i < 11`, but a YYYY-MM-DD date is 10
characters. The case was rewritten from scratch (same error string,
same range checks y 1970–2099, mo 1–12, d 1–daysIn): explicit length-10
check, `'-'` at positions 4 and 7, digits elsewhere, then the same
range test. `valueOf`, `'date'` case, solve.js lines 219–238.

### Round 2 audit — every length / index / slice bound re-checked

Each item below was compared against the reference libs
(`frontmatter.js`, `markdown.js`, `template.js`, `text.js`, `index.js`
in the kit's `baselines/makefun-site/vendor/makefun-site/`). No other
genuine mismatch was found; nothing else was changed.

- `longDate` slices (0,4), (5,7), (8,10) — right for a 10-char date.
- `slugifyRaw`: cut at 64, then trailing `-`s stripped — matches
  `SLUG_MAX` + `replace(/-+$/, '')`.
- `stringOf`: loop `i < raw.length - 1`, closed-check `length < 2` —
  matches the reference.
- `tags` value: `slice(1, -1)`; `tagOk` matches `TAG` (`[^"[\]]+`).
- `slug` value: `raw.length <= 64` — matches the reference.
- `orderedMarker`: at most 9 digits (`d < 9`) — matches `(\d{1,9})`.
- `headingOf`: 1–6 `#`s, space/tab after, trailing spaces/tabs
  trimmed, empty text rejected — matches `HEADING`.
- `isHr`: length ≥ 3, `---` or `***` then only spaces/tabs — matches `HR`.
- `fenceLang` / `isFenceClose`: backticks at 0–2, lang chars
  `[A-Za-z0-9_+-]`, trailing spaces/tabs only — matches `FENCE_OPEN` /
  `FENCE_CLOSE`.
- `isContinued`: ≥ 2 literal spaces then a non-whitespace char —
  matches `^ {2,}\S` (the `isWsCode` set is exactly JS `\s`).
- `bulletMarker` / `orderedMarker` / `listFirst`: `- ` / `* ` /
  `N. ` shapes — match `BULLET` / `ORDERED`.
- `findUnescaped`: bound `j <= s.length - what.length` — matches.
- `findSingleStar`: same loop as the reference.
- Inline code/strong/em/link slices (`j+1..m`, `j+2..m`, `kk+2..m`,
  `j+1..kk`) — match the reference's index arithmetic.
- Blockquote strip: `>` plus one optional space (`slice(1)` /
  `slice(2)`) — matches `replace(/^> ?/, '')`.
- `tokenize`: `open === src.length - 1` guard, `+2` / `+3` close
  offsets, standalone-tag line handling — match the reference.
- `flattenOps` jump targets (`els`, `end` for if/else and for) and
  `renderRange`'s jumps — re-walked by hand; the `k === 3` op lands
  on `op.end` exactly as the reference's tree walk returns.
- `resolvePath` / `dirOf`: `lastIndexOf('/')`, `..` popping with
  null on climb-out — match the reference.
- `hasScheme`: first char a letter, then alphanumerics / `+ - .`,
  `:` ends it — matches `/^[A-Za-z][A-Za-z0-9+.-]*:/`.
- `parseFM`: close-line scan from 1, `line N` numbering (`i + 1`),
  `key: value` shape (colon + single space), trailing space/tab
  strip, unknown-key and duplicate checks in the reference's order.
- `stripTrailingSlashes`: strips trailing `/`s — matches
  `base_url.replace(/\/+$/, '')`.
- `cardOf` reading minutes: `Math.max(1, Math.ceil(words / 200))`.
- Pagination slices `(k-1)*per .. k*per`, index/tag URL shapes.
- Include depth: render only while `depth < 8` — matches
  `INCLUDE_DEPTH`.
- `uniqueSlug`: suffixes from `-2` — matches.
- `lookup` scope walk, `textOf` array join `", "`, `truthy` — match.

### Speed changes (no semantic change intended)

- `esc` (solve.js line 41): fast path is now one regex test
  (`/[&<>"']/.test(s)`) instead of five `indexOf` scans; the
  replacement loop is untouched.
- `inline` (solve.js lines 428, 431–519): one module-level regex
  (`SPECIAL_RE`, `/[\\`*\[]/g`, `lastIndex` set before every exec)
  finds the next special in a single scan instead of four `indexOf`
  passes; output accumulates in one string (`out +=`) instead of a
  `parts` array + `join`; an escaped character is emitted directly
  (`'>'` → `'&gt;'`, anything else unchanged — `'>'` is the only
  escapable character the HTML escaper touches) instead of an `esc()`
  call.
- `wordsOf` (solve.js lines 116–127): manual whitespace-transition
  scan instead of `text.match(/\S+/g)` — same count, no match array
  allocated. Relies on `isWsCode` being exactly the JS `\s` set
  (verified in the audit above).
- `renderMd` return (solve.js line 611): `blocks.join('\n') + '\n'`
  instead of `blocks.map(b => b + '\n').join('')` — identical output,
  no intermediate array.
- `renderRange` var branch (solve.js lines 791–796): a string value
  skips the `textOf()` call (`textOf` returns strings unchanged).
- NOT changed: `slugify`/`slugifyRaw` (low profile share, and the
  for-of loop is what keeps astral-plane text correct), `lookup`,
  `findUnescaped`/`findSingleStar`, the `for` vars-object reuse, the
  template compiler, and the `run()` pipeline order. No caches were
  added across `run()` calls — cases stay independent.

### Corners still unsure (for parent verification)

- The `SPECIAL_RE` optimisation is the one change whose speed I
  could not measure (no node run in this revision): a global regex
  with `lastIndex` reset before every `exec` is safe under the
  synchronous, single-threaded rendering here (including the
  recursive `inline` calls, which reset it themselves), but the
  parent's timing run is the real verdict — if warm time does not
  improve, reverting to the four `indexOf` calls loses nothing
  semantically.
- `wordsOf`'s identity with `/\S+/g` rests on the `isWsCode` set
  matching JS `\s` exactly; it was checked character class by
  character class, but a fuzz case with exotic whitespace (e.g.
  U+1680, U+200B — note U+200B zero-width space is NOT `\s`, and
  `isWsCode` correctly excludes it) would settle it.
