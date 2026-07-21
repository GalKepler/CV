# CV repo architecture

## Reality vs. the brief's assumption

The improvement brief assumed a pandoc + Rmd build pipeline (Makefile/GH Action
invoking pandoc with a template). That pipeline **used to exist** — `git log`
shows `index.Rmd` and `resume.Rmd` as the original sources, rendered via the
`vitae`/`pagedown` R packages (`pagedown::html_resume` template, hence the
`paged-x.x` JS/CSS folders and `<meta name="generator" content="pandoc">` tag
that's still in `index.html`).

At some point the `.Rmd` sources were deleted from the repo. **What's left is
only the rendered static output**, which has since been hand-edited directly
(see the run of "trying to fix publications rendering" commits). There is no
build step, no template, no CI, and no `Makefile` — `index.html` and
`my_css.css` *are* the source of truth now.

## What's actually in the repo

| File | Role |
|---|---|
| `index.html` | The entire CV. Static HTML, hand-edited. Content lives directly in the markup — there is no separate data file. |
| `my_css.css` | All styling, including the `paged.js`-based print/pagination layout (timeline dates, decorator line, sidebar). |
| `index_files/`, `resume_files/` | Vendored JS/CSS assets from the original pagedown/vitae template: Font Awesome, `paged.js` (a polyfill that paginates HTML into CSS `@page`-sized boxes, live in the browser, for print/PDF preview). |
| `print.R` | Leftover from the original template fork — still references `pagedown::chrome_print("index.html", output = "Mattan-Ben-Shachar_CV.pdf")`, i.e. someone else's name. Not wired into anything; not run by CI. |
| `count_students.xlsx` | Raw teaching-load data (course/year/headcount). Used manually to compute the student-count figures now in the CV; not read at render time. |
| `.nojekyll` | Tells GitHub Pages to serve the repo root as-is (static site, no Jekyll processing). |

There is no `.github/workflows/` — deployment is just "whatever's on the
default branch is what GitHub Pages serves from the repo root."

## How the page renders

1. `index.html` loads Font Awesome + `paged.js` (only on viewports
   ≥768px — see below) + `my_css.css`.
2. An inline `<script>` at the bottom of `index.html` does DOM surgery: it
   turns the raw `# Aside` / `# Main` pandoc-style headings and flat
   `<p>` sequences (org / location / date, in that fixed order) into the
   timeline layout (date column, decorator line/dot, details block). This is
   position-based — it assumes exactly 3 leading `<p>` tags per entry
   (org, location, date) — so changing that shape requires updating both the
   markup and the script in tandem.
3. On viewports ≥768px, `paged.js` then paginates the whole thing into
   fixed A4-sized `.pagedjs_page` boxes, which is what makes the on-screen
   preview look like a stack of printable pages. Below 768px, `paged.js` is
   skipped entirely (see "Responsive" below) and the plain unpaginated flow
   renders instead, styled by a dedicated mobile CSS block.
4. A small inline script fetches `https://api.github.com/repos/GalKepler/CV/commits?path=index.html`
   at view time and stamps the sidebar "Last updated" date from the most
   recent real commit — there's no build step to do this at build time, so
   this is the closest equivalent for a static, buildless repo.

## Known limitation: PDF export

`paged.js`'s pagination is a live JS/DOM effect. Chrome's native
"print to PDF" (what `pagedown::chrome_print` in `print.R` wraps) needs to
wait for `paged.js` to finish laying out pages before printing, or it
captures a stale, unpaginated single page. `pagedown::chrome_print` handles
this wait internally; a bare `chrome --headless --print-to-pdf` does not.
`print.R` itself is stale (wrong filename, not run by anything) — if a PDF
export needs to be produced regularly, it should become a real CI step
rather than a local manual script.

## Deliberately not done in this pass

- **No `data/cv.yaml` + template rewrite.** The brief's §5 structured-data
  proposal is real, but it means rebuilding the render pipeline from
  scratch (there's no template to point Jinja/pandoc at — it was deleted).
  That's a bigger, separate undertaking, not a long-tail bug fix.
- **No industry/academic/ATS variant builds** (§3, §5.4) — same reason,
  plus needs the structured-data step first.
- **No CI, lint script, ORCID sync, or Lighthouse/axe gates** (§5.3, §6) —
  there's nothing to run these against yet without a build step; a CI
  workflow that just deploys static files as-is would add process without
  catching the class of bugs (`N/A` leakage, stale dates) the brief cared
  about, which were fixed directly at the source this round instead.
- **The timeline (left-column date + decorator line) layout was kept**
  rather than switched to the brief's suggested "date on the same line as
  the role, right-aligned" two-line format. Ripping out the decorator/date
  positioning system is a bigger, riskier change to the fragile
  position-based JS than the stylistic gain justifies; the existing
  timeline already reads as a clean, professional layout.
