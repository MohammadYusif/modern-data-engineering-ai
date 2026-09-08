# modern-data-engineering-ai — Quarto course-site rules

This repo publishes a five-day course as a [Quarto](https://quarto.org)
website to GitHub Pages, built from the course's original slide decks
(`.pptx`, not tracked in this repo) and lab notebooks. Each day gets its own
`L0N/` folder: a prose lesson (`.qmd`) per topic plus the day's lab
notebook(s), rendered directly into the site (see `_quarto.yml`'s `render:`
list) — unlike `agentic-ai-systems`, there is no separate Colab-twin
`notebooks/` folder here; the lab notebook shown on the site *is* the lab.

This file carries the lessons learned building `agentic-ai-systems` (see
that repo's own `CLAUDE.md`) plus one more learned building this repo's
Day 1. Treat it as binding for every day added here.

## Before touching anything: render, don't just read source

A `.qmd`/`.ipynb` file looking correct in the editor proves nothing. Quarto's
pandoc pass changes markdown structure in ways invisible in source and only
visible in the rendered HTML. **Always verify against rendered output**, not
the source file:

```
quarto render
quarto preview
```

Then check the actual DOM — links resolve, tables render, mermaid diagrams
became real `<svg>` (not raw `graph LR` text left unrendered), no console
errors, no broken images.

## Verify every lab notebook cell was actually executed

A notebook can read as complete — functions defined, a `main()` call at the
bottom — while a whole section was never actually run. This happened with
Day 1's `Day1_Lab.ipynb`: Part 2 (Spark/Delta) had real captured output, but
Part 1 (an ETL/ELT/Lakehouse comparison, pure pandas) had `execution_count:
null` and zero saved outputs on every one of its cells. It looked fine on
read-through and would have shipped un-executed if not specifically checked.

Before treating any lab notebook — the original source file or the copy
going into this site — as done, check every code cell:

```
python -c "import json; nb=json.load(open('path.ipynb', encoding='utf-8')); [print(i, c.get('execution_count'), len(c.get('outputs',[]))) for i,c in enumerate(nb['cells']) if c['cell_type']=='code']"
```

Any code cell showing `execution_count: None` with 0 outputs needs attention:

- **If it can run in this environment** (pure Python/pandas, no exotic
  infra/cloud dependency) — actually run it and inject the real captured
  output into that cell, in **both** the site's copy under `L0N/` and the
  original source lab notebook under `ModernDataEngineering/<N>_Day .../Day
  N Lab/`, so the source itself stays fixed, not just what's published.
- **If it genuinely needs infrastructure this environment doesn't have** — a
  live Kafka broker, a cloud vector DB (Pinecone/Weaviate), Airflow, a GPU —
  don't fabricate output. Say so, and add a `::: {.callout-note}` in the
  lesson page pointing at Colab or whatever environment it actually needs
  (see the Windows/PySpark caveat in `L01/01_lakehouse_architectures.qmd`
  for the pattern).

This is the same bar the workspace's own grading rules apply to student
submissions once grading is automated — an unexecuted "proof" notebook with
no captured outputs is a hold, not a pass. Hold our own published labs to it
too.

## A markdown cell that starts with a bare `---` breaks the whole notebook render

Found building Day 3's `Day3_Exercise.ipynb`: several markdown cells used a
literal `---` horizontal rule as a section divider before each Task heading:

```md
---
## Task 1 — Document Chunking (10 min)
...
```

Quarto's ipynb-to-document conversion also uses `---` as a separator when
assembling cells into one document, so a markdown cell whose content *starts*
with `---` gets misread as a stray YAML block opening mid-document. The
render fails with a `YAMLException` pointing at content several cells later
(in this case, `"end of the stream or a document separator is expected"`,
with the error location showing unrelated prose from deep in the cell) —
the error message doesn't point at the actual `---` that caused it, which
makes this easy to misdiagnose as a problem in whatever content the error
message happens to quote.

Fix: never start a markdown cell's content with a bare `---`. Drop it (a
`##` heading is section-break enough) or move it so it isn't the cell's
first line. Check every markdown cell in a notebook headed for this site,
not just the one the error message blames:

```
python -c "import json; nb=json.load(open('path.ipynb', encoding='utf-8')); [print(i, repr(''.join(c['source'])[:10])) for i,c in enumerate(nb['cells']) if c['cell_type']=='markdown']"
```

Any cell whose printed prefix starts with `'---` needs fixing.

## A markdown cell's `source` MUST be a list of lines, not one flat string

The nastiest one so far. Found across `Day3_Lab.ipynb`, `Day3_Exercise.ipynb`,
and `Day5_Lab.ipynb` after tools (a scripted frontmatter-insertion, and this
repo's own `NotebookEdit`-based fixes) wrote a markdown cell's `source` field
as one single string instead of nbformat's normal list of per-line strings.
A cell like:

```json
"source": "## Task 1 — Document Chunking\n\nSplit each document into overlapping chunks.\n\nRules:\n..."
```

renders as a **single `<h2>` that swallows the entire rest of the cell** —
the heading, every following paragraph, list, and blockquote all end up
inside the one `<h1>`/`<h2>`/`<h3>` tag (visible as a monstrous `data-anchor-id`
slug containing the whole cell's text). Quarto/pandoc's ipynb reader handles
this fine when `source` is the standard list-of-lines form:

```json
"source": ["## Task 1 — Document Chunking\n", "\n", "Split each document into overlapping chunks.\n", "\n", "Rules:\n", "..."]
```

There's no error, no warning — the page renders "successfully" with a
badly broken heading. This is easy to miss by eye (it can look fine
skimming the rendered page) but shows up immediately if you check heading
length:

```
python -c "import json; nb=json.load(open('path.ipynb', encoding='utf-8')); print([len(''.join(c['source']) if isinstance(c['source'], list) else c['source']) for c in nb['cells'] if c['cell_type']=='markdown'])"
```

Any markdown cell whose `source` is a plain `str` (not a `list`) is a
candidate — check it, or just normalize every markdown cell before shipping:

```python
for cell in nb["cells"]:
    if cell["cell_type"] == "markdown" and isinstance(cell["source"], str):
        cell["source"] = cell["source"].splitlines(keepends=True)
```

**This means: after using `NotebookEdit` (or any script) to change a markdown
cell's content, re-check that cell's `source` is still a list, not a flattened
string** — `NotebookEdit` and ad-hoc frontmatter-insertion scripts have both
produced the flat-string form in practice. Verify against the rendered
heading (see "before touching anything: render, don't just read source"
above), not just that the notebook JSON is syntactically valid.

## The Colab-badge trap (pandoc implicit-figures) — if badges are ever added

Not currently used in this repo, but binding if a Day page ever links to a
standalone Colab copy of its notebook. Never write a linked image as its own
markdown paragraph:

```md
[![Open In Colab](badge.svg)](https://colab.research.google.com/...)
```

Quarto's implicit-figures pass silently drops the enclosing `<a>` when a
linked image is the only thing in its paragraph — the image still renders,
but the click target is gone. Write badges as raw HTML instead:

```html
<a href="https://colab.research.google.com/github/<user>/<repo>/blob/master/<path>.ipynb" target="_blank" rel="noopener"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"></a>
```

## Links: audit the rendered site, not just grep source

Before publishing, and periodically afterward (upstream docs move), audit
every `<a href>`/`<img src>` on every rendered page — internal targets exist,
fragments resolve, external URLs return non-error status.

- Content adapted from another vendor's docs (Databricks, Delta Lake,
  Confluent, LangChain, etc.) can carry root-relative links that resolve on
  *their* domain and go nowhere on ours — rewrite to the full external URL.
- Before "fixing" a flagged link, confirm it's real — a string that looks
  like a link inside a captured cell **output** (not a real `<a href>` in
  the rendered page) is a false positive.

## `_quarto.yml` gotchas

- **Don't add `revealjs` as a project-level format.** Listing it alongside
  `html` makes Quarto render every page twice, both claiming the same
  `<name>.html` — the whole project render dies with a "NotFound ... rename"
  race. If a page needs slides, render it explicitly:
  `quarto render <page>.qmd --to revealjs`, with slide options in that
  page's own front matter.
- **`execute: enabled: false` at the project level** is deliberate — lab
  notebooks ship with saved outputs from a real executed run; rendering must
  not try to re-execute them (no local Spark/Kafka/vector-DB environment
  here). This makes the "verify every cell was actually executed" rule above
  load-bearing: if a cell's output was never captured, it renders as empty,
  not as an error — the gap is silent unless you check for it.
- **New day folders need adding to `render:`** — the glob `"L0*/*.ipynb"`
  covers new `L0N/` folders automatically, but double-check a new day's
  notebooks actually match before assuming they're included.
- When editing this file with a scripted tool (`sed`, bulk find/replace), a
  partial edit can strand orphaned child keys under a since-deleted parent
  block. After any scripted YAML edit, re-read the whole block, not just the
  line you targeted.

## A lab that writes local persistent state may leak it on Windows, not on Colab

Found adding L03's `PersistentClient`/metadata-filtering demo: `del client, collection`
plus `gc.collect()` does **not** reliably release a SQLite file handle on Windows,
so `shutil.rmtree(persist_dir, ignore_errors=True)` silently fails to remove the
directory afterward (the `ignore_errors=True` masks it completely — no exception,
just a leftover folder). This is Windows-specific: POSIX (Colab, Linux, macOS —
what every student actually runs on) allows unlinking a directory entry while a
process still holds a file inside it open, so the exact same cell cleans up fine
there. Re-running the cell doesn't break correctness (verified: a second run with
the stale directory still present produces the same correct output), so this
isn't a bug students will ever see — but it does mean **testing a lab that writes
local persistent files on this Windows trainer machine will leave real
directories on disk that must not get committed.** Add anything a lab writes to
disk (a Chroma/Delta/Kafka-log directory, a checkpoint folder) to `.gitignore`
defensively, and don't trust `del` + `gc.collect()` as proof of cleanup on
Windows — check with `ls`/`Test-Path` after running, not just by reading the code.

## Keep the `.qmd` lesson and its lab notebook in sync

When actually running a lab notebook surfaces something the slide-derived
prose got wrong or left out — a version pin, an API that changed, a
workaround an environment needed — port that fix back into the paired
lesson page too, so "read it" and "run it" don't diverge.

## General rule

Before calling a day "done": re-render clean, check every lab notebook cell
has real captured output (not just that it looks right in source), and check
links/tables/diagrams in the actual rendered HTML — not the editor.
