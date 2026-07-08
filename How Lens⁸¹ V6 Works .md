# Lens⁸¹ v6 — "Highlight → Find Citation" Feature Documentation

This document explains, in detail, how the new citation-finder feature works
under the hood. It covers the full pipeline, every file involved, the exact
message contract between the content script and the background service
worker, and the design decisions (especially the zero-LLM-cost constraint)
behind them.

---

## 1. What it does, from the user's side

1. You highlight a sentence or two — on any webpage, or inside a Google Doc.
2. A small **📎 Cite** button appears near your selection.
3. Click it. It searches three free academic databases for papers that share
   vocabulary with what you highlighted.
4. A popover shows up to 5 candidate papers, each with a relevance score and
   the specific words that matched.
5. Pick a citation style (APA / MLA / IEEE / BibTeX) from a dropdown per
   result, then either:
   - **Insert** — drops the formatted citation at your cursor, or
   - **📋** — copies it to your clipboard.

No API key, no LLM call, no cost — the entire feature runs on free,
keyless search APIs plus local JavaScript.

---

## 2. High-level architecture

```
                         YOU highlight text
                                │
                                ▼
                    cite.js (content script)
                    injected on <all_urls>
                                │
              detects selection (or, on Google Docs,
              captures it via a native copy/clipboard read)
                                │
                                ▼
                  floating "📎 Cite" button appears
                                │
                          you click it
                                │
                                ▼
         chrome.runtime.sendMessage({ type: 'FIND_CITATIONS', text })
                                │
                                ▼
                background.js (service worker)
                   'FIND_CITATIONS' message handler
                                │
                    ┌───────────┴────────────┐
                    │   1. check local cache  │
                    │      (30-day TTL)       │
                    └───────────┬────────────┘
                          cache miss
                                │
                                ▼
                 2. extract search keywords LOCALLY
                    (stopword-strip + longest-words-first
                     — no model call)
                                │
                                ▼
        3. search THREE free APIs in parallel, no API key needed:
           ┌───────────────┬───────────────────┬───────────────┐
           │   Crossref    │  Semantic Scholar  │    OpenAlex   │
           └───────┬───────┴─────────┬─────────┴───────┬───────┘
                   └─────────────────┼─────────────────┘
                                     ▼
                  merge + de-duplicate by title similarity
                                     │
                                     ▼
        4. score each candidate LOCALLY: % of your passage's
           distinctive words that appear in its title/abstract
           (no AI judgment — plain word-overlap arithmetic)
                                     │
                                     ▼
              drop anything below a 20% overlap threshold,
                keep the top 5 by score
                                     │
                                     ▼
        5. format each survivor into APA / MLA / IEEE / BibTeX
           strings using hand-written templates (no citeproc-js)
                                     │
                                     ▼
                     write result to 30-day cache
                                     │
                                     ▼
              sendResponse(...) back to cite.js
                                     │
                                     ▼
                 popover renders the ranked results
                                     │
                          you click Insert / Copy
                                     │
                    ┌────────────────┴────────────────┐
                    │  regular page: restore saved     │
                    │  selection Range, execCommand    │
                    │  ('insertText') or direct         │
                    │  textarea/input manipulation      │
                    ├───────────────────────────────────┤
                    │  Google Docs: write to clipboard, │
                    │  refocus the doc's hidden input   │
                    │  iframe, execCommand('paste')      │
                    │  (allowed because the extension    │
                    │  has clipboardRead/Write perms)    │
                    └───────────────────────────────────┘
                    (clipboard is ALWAYS written first,
                     as a guaranteed Ctrl/Cmd+V fallback)
```

---

## 3. Files involved

| File | Role | Touched how |
|---|---|---|
| `manifest.json` | Declares the new content script, new permissions | Additive edit |
| `background.js` | Runs the entire search → score → format pipeline | Additive section appended at the end |
| `cite.js` | **New file.** All UI: selection detection, button, popover, insert/copy | New |
| `styles.css` | Visual styling for the button/popover | Additive section appended at the end |

Nothing about the existing classification pipeline, Collections, popup, or
options page was modified to build this — see §9 for exactly how that
isolation is enforced.

### 3.1 `manifest.json` changes

```jsonc
"permissions": ["storage", "activeTab", "clipboardRead", "clipboardWrite"],
```
`clipboardRead`/`clipboardWrite` are what let the Google Docs copy/paste
trick work at all (see §6) — ordinary web pages can't use
`document.execCommand('paste')`, but a content script with these
permissions declared can.

```jsonc
"host_permissions": [
  "https://api.semanticscholar.org/*",
  "https://api.openalex.org/*",
  "https://api.crossref.org/*",     // new
  ...
]
```
`api.crossref.org` was added so the background service worker's fetch calls
to Crossref reliably bypass CORS, the same way the existing Semantic
Scholar/OpenAlex calls already do.

```jsonc
"content_scripts": [
  { "matches": ["https://scholar.google.com/*"], ... },   // existing, untouched
  {
    "matches": ["<all_urls>"],
    "css": ["styles.css"],
    "js": ["cite.js"],
    "run_at": "document_idle"
  }
]
```
A **second, separate** content-script block. The existing Scholar-only
block wasn't edited — this is why Chrome now shows the broader "read and
change your data on all websites" permission warning: the new block matches
every page, not just Scholar.

---

## 4. The message contract

One new message type, `FIND_CITATIONS`, handled by its own
`chrome.runtime.onMessage.addListener(...)` call in `background.js` — a
second listener, not a branch inside the existing one, so the original
listener and every message type it already handles are untouched.

**Request** (from `cite.js`):
```json
{ "type": "FIND_CITATIONS", "text": "<highlighted passage>" }
```

**Success response:**
```json
{
  "ok": true,
  "source": "keyword-match",
  "keywords": ["deep", "learning", "cancer", "detection"],
  "results": [
    {
      "title": "Deep Learning Approaches for Cancer Detection in Medical Imaging",
      "authors": ["John Smith", "Alice Doe"],
      "year": 2023,
      "venue": "Nature Medicine",
      "doi": "10.1000/xyz123",
      "url": "https://doi.org/10.1000/xyz123",
      "relevance": 75,
      "why": "Shares keywords: deep, learning, cancer, detection, medical",
      "citations": {
        "apa": "Smith, J., & Doe, A. (2023). Deep Learning Approaches...",
        "mla": "Smith, John, et al. \"Deep Learning Approaches...\"",
        "ieee": "[1] J. Smith et al., \"Deep Learning Approaches...\"",
        "bibtex": "@article{Smith2023Deep,\n  author = {...},\n  ...\n}"
      }
    }
  ]
}
```

**Failure response:**
```json
{ "ok": false, "error": "no-results" }
```
Possible `error` values: `empty`, `no-results`, `no-strong-match`, `timeout`,
`unavailable`.

All four citation formats are pre-computed server-side (in the background
worker), so switching format in the popover's dropdown is instant and never
needs another round-trip.

---

## 5. The pipeline, step by step

### Step 1 — Cache lookup
`findCitations(rawText)` trims the text, truncates it to 800 characters if
longer (a whole paragraph is fine; anything past that is just bounded, not
rejected), and computes a cache key:

```
cacheKey = 'cite:' + citeHash(normalize(claimText))
```

`citeHash` is a small non-cryptographic hash (djb2 algorithm) — its only
job is keeping the `chrome.storage.local` key short and collision-unlikely
for a purely local cache; it's not a security boundary. Cached results are
valid for **30 days**, the same TTL the classification cache already uses
(the constant is literally reused: `CITE_CACHE_TTL_MS = CACHE_TTL_MS`).
On a cache hit, the pipeline stops here — no network requests at all.

### Step 2 — Local keyword extraction
`naiveKeywords(text)`:
1. Lowercases and strips punctuation.
2. Removes a fixed stopword list (articles, conjunctions, pronouns, etc.).
3. De-duplicates what's left.
4. Sorts by **word length**, longest first — a cheap proxy for
   "distinctive/technical" in the absence of any model to judge actual
   importance.
5. Takes the top 6.

These 6 keywords are joined into a single search-engine query string.

### Step 3 — Parallel search across three free APIs
```js
const [cr, ss, oa] = await Promise.allSettled([
  searchCrossref(query),
  searchSemanticScholar(query),
  searchOpenAlex(query),
]);
```
Each returns up to 8 candidates, normalized to a common shape:
`{ title, abstract, authors, year, venue, doi, url }`.

| Source | Endpoint | Notes |
|---|---|---|
| **Crossref** | `api.crossref.org/works?query=...` | Broadest DOI/metadata coverage, especially outside CS/AI. Abstracts (when present) come as JATS-tagged XML and are stripped of tags. |
| **Semantic Scholar** | `api.semanticscholar.org/graph/v1/paper/search` | Strong for CS/AI papers; same endpoint the existing abstract-lookup feature already uses. |
| **OpenAlex** | `api.openalex.org/works?search=...` | Very broad coverage; abstracts arrive as an "inverted index" and are reconstructed into plain text by the existing `reconstructAbstract()` helper. |

`Promise.allSettled` means one source timing out or erroring doesn't sink
the other two — each has its own 8-second timeout (`CITE_SEARCH_TIMEOUT_MS`).

### Step 4 — De-duplication
`dedupeCandidates(list)` folds together results whose titles are highly
similar (using the same `titleSimilarity()` heuristic the existing
abstract-lookup feature relies on, threshold > 0.85). When two sources
return the "same" paper, whichever copy is missing an abstract inherits the
other's — abstracts matter here because…

### Step 5 — Local relevance scoring (no AI)
This is the part that replaced the original LLM re-rank call. For each
candidate:

```js
function localRelevance(claimText, candidate) {
  const claimWords = <distinctive words in the highlighted passage>;
  const candidateWords = <words in candidate.title + candidate.abstract>;
  const matched = claimWords ∩ candidateWords;
  const score = round(matched.length / claimWords.size * 100);
  return { score, matched };
}
```

In plain terms: *what percentage of the meaningful words in your
highlighted passage also appear in this paper's title or abstract?* That
percentage **is** the relevance score shown to you, and the matched words
**are** the "why" shown under each result (e.g. *"Shares keywords: deep,
learning, cancer, detection, medical"*) — there's no black box here; the
explanation is the literal computation.

Candidates scoring below **20%** (`CITE_LOCAL_RELEVANCE_THRESHOLD`) are
dropped entirely rather than shown as a weak match. The remaining
candidates are sorted by score, descending, and capped at **5** results
(`CITE_MAX_RESULTS`). If nothing survives, the response is
`{ ok: false, error: 'no-strong-match' }` — an honest "nothing found," not a
forced weak suggestion.

### Step 6 — Citation formatting
Each surviving candidate is formatted into all four styles up front:

- **APA** — `Last, F. I., & Last2, F. I. (Year). Title. Venue.`
- **MLA** — `Last, First, et al. "Title." Venue, Year.`
- **IEEE** — `[n] F. Last et al., "Title," Venue, Year.` (n = position in
  this result list — see limitation below)
- **BibTeX** — `@article{LastYearFirstword, author = {...}, title = {...}, ...}`

These are **hand-written template functions**, not a CSL (Citation Style
Language) processor like `citeproc-js`. Author name splitting is a simple
heuristic (last word of a full name = surname, everything before it =
given-name initials) — it won't handle every name format perfectly (e.g.
names with multiple surnames, or given-name-first cultural naming
conventions), the same honesty-over-completeness trade-off already made for
the existing BibTeX export in Collections.

### Step 7 — Cache and return
The full response is written to the 30-day cache under the computed key,
then sent back to `cite.js` via `sendResponse()`.

---

## 6. Getting and inserting text: regular pages vs. Google Docs

This is the trickiest part of the feature, because **Google Docs doesn't
behave like a normal webpage.**

### Regular webpages
- **Reading the selection:** `window.getSelection()` gives real text
  directly. On `mouseup` (and on relevant `keyup` events, for
  keyboard-driven selection like Shift+Arrow), `cite.js` saves both the
  selected text and a cloned `Range` object.
- **Showing the button:** positioned at the selection's bounding rectangle.
  Its `mousedown` handler calls `e.preventDefault()` — without this, simply
  clicking the button would collapse the browser's text selection before
  the click handler even runs.
- **Inserting a citation:**
  - If focus is in a `<textarea>` or `<input>`, the citation is spliced
    directly into `.value` at the saved cursor position (more reliable
    across browsers than `execCommand` for these elements).
  - If focus is in a `contenteditable` region, the saved `Range` is
    restored to the page's selection (clicking into the popover moved focus
    away from it), then `document.execCommand('insertText', false, text)`
    drops it in.
  - If the highlighted text isn't inside anything editable at all (e.g. you
    highlighted a sentence on a news article you're reading, not writing),
    there's no cursor to insert at — the citation was already written to
    the clipboard, so the person just gets a "Copied" status instead of a
    false "Inserted" claim.

### Google Docs
Google Docs renders its content on `<canvas>`/SVG, with actual text input
routed through a hidden `contenteditable` iframe
(`.docs-texteventtarget-iframe`). This means:
- `window.getSelection().toString()` **does not return the real highlighted
  text** — it reflects the invisible iframe, not what you see on screen.
- A content script can't just call `execCommand('insertText')` on the
  visible document the way it can on a normal page.

The workaround, used by a number of real-world Docs-integrating extensions:

**Reading the selection** (`citeGetSelectedText()` when `CITE_IS_DOCS` is
true):
```js
document.execCommand('copy');              // triggers Docs' own copy handling
await sleep(60);                            // let the clipboard write land
const text = await navigator.clipboard.readText();
```
This works because `execCommand('copy')` fires a real, native `copy` event
— the same one a physical Ctrl+C would — which Google Docs' own JavaScript
intercepts and handles by writing the actual selected content to the OS
clipboard. Reading it back gives the real text.

**Inserting the citation** (`citeInsertIntoDocs()`):
```js
await navigator.clipboard.writeText(citationText);
focusDocsEditor();                          // refocus the hidden input iframe
document.execCommand('paste');
```
`execCommand('paste')` is blocked for ordinary web pages by all modern
browsers — but a content script whose extension has declared the
`clipboardRead`/`clipboardWrite` permissions (which this one now does) is
allowed to invoke it. Docs' own paste handling then reads the clipboard and
inserts the content at the cursor.

**Two honest caveats, documented for the user in both the button's tooltip
and the README:**
1. Using Cite in a Google Doc **will overwrite your OS clipboard** with
   whatever you highlighted, then again with the citation you insert.
2. This entire mechanism rides on Google Docs' internal copy/paste
   handling rather than a documented, stable API — it's inherently more
   fragile than everything else in the extension. That's why the citation
   is **always** written to the clipboard first, regardless of platform:
   if the programmatic paste silently fails, Ctrl/Cmd+V is a guaranteed
   fallback.

---

## 7. Caching

| | Classification cache | Citation cache |
|---|---|---|
| TTL | 30 days | 30 days (same constant, reused) |
| Key prefix | (paper-specific) | `cite:` |
| Validation on read | Requires `reason` + `details` fields | No such requirement |
| Functions | `getCache()` / `setCache()` | `getCiteCache()` / `setCiteCache()` |

The citation cache **deliberately does not reuse** `getCache()`/`setCache()`
even though the TTL logic is identical — the existing functions validate
that a cached value has classification-specific fields (`reason`,
`details`). Reusing them as-is for citation data (which has neither field)
would make every citation lookup silently look like a permanent cache miss.
Separate functions, same pattern, zero shared state.

---

## 8. Why no LLM at all (by design, not by fallback)

An earlier iteration of this feature used an LLM to extract keywords and
re-rank candidates when a model was configured, falling back to
keyword-only search otherwise. That version was replaced entirely — the
current pipeline **never** calls `getConfiguredPairs()` or
`callProviderChat()`, regardless of what's set up in Settings. This was a
deliberate trade-off:

**What you gain:** the feature costs nothing beyond three free API calls,
no matter how many or how expensive your configured model rows are for
classification. It also can't produce an AI hallucination about relevance —
the "why" is always a literal, checkable list of shared words.

**What you give up:** word-overlap scoring can't recognize synonyms
(“neural network” vs. “deep learning” won’t match each other) and can't
distinguish a paper that genuinely discusses your claim from one that
merely happens to share technical vocabulary in an unrelated context. In
practice, expect more genuine misses (nothing found) than an AI-ranked
version would produce — but everything that *is* surfaced should share real
vocabulary with your highlighted passage, not just a model's opinion that it
does.

If AI-boosted ranking is wanted later, it would be a separate, explicitly
opt-in addition — not something the default pipeline does.

---

## 9. How this stays isolated from the rest of the extension

Every design choice below was made specifically so this feature could be
added without risking the existing classification/Collections features:

- **A second `chrome.runtime.onMessage` listener**, not a new branch in the
  existing one. Chrome supports multiple listeners; each receives every
  message independently, so the original listener's `if` chain for its own
  message types was never touched.
- **A second, separate `content_scripts` block** in `manifest.json`,
  leaving the original Scholar-only block untouched.
- **A dedicated cache (`getCiteCache`/`setCiteCache`)** instead of reusing
  the classification cache's read/write functions, which have
  classification-specific field validation baked in.
- **A brand-new file, `cite.js`**, rather than adding logic to
  `content.js` or `collections-content.js` — those files run only on
  `scholar.google.com` and were not modified at all.
- **Pure appends** to `background.js` and `styles.css` — everything above
  the new section in each file is byte-for-byte identical to v5.

---

## 10. Known limitations (summary)

- Word-overlap relevance can miss genuinely relevant papers that use
  different terminology (synonyms), and can't verify that a shared term is
  used in the same *sense* as your passage.
- Author-name splitting for citation formatting is a heuristic (last word =
  surname), not a real bibliographic name parser.
- IEEE numbering (`[1]`, `[2]`, …) reflects only this result set's order —
  it won't stay consistent with other citations you've already inserted
  elsewhere in the same document.
- The Google Docs copy/paste mechanism depends on Docs' internal behavior
  rather than a documented API, and may need adjustment if Google changes
  how Docs handles copy/paste internally.
- Crossref abstracts are only present for a subset of records; when no
  source has an abstract for a given paper, relevance scoring falls back to
  title-only overlap, which is a weaker signal.
