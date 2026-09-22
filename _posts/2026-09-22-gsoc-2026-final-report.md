layout: post

title: "GSoC 2026 Final Report: SAGE - LLM-Assisted Extraction of Agronomic and Ecological Experiments into Structured Data"

date: 2026-09-22 00:00:00 +0000

author: Abhishek Kumar Rai

tags: [GSoC 2026, PEcAn, LLM Agents, Python, FastAPI, Pydantic, Streamlit, Data Extraction]

excerpt: >-

  Google Summer of Code 2026 final report — building SAGE, an LLM pipeline that turns published crop-science

  papers into a structured, source-grounded, human-reviewed intermediate representation for the PEcAn project.

cover: /assets/images/gsoc_header.png

<style>
/* ==========================================================================
   SAGE GSoC 2026 Final Report — presentation layer
   This block is intentionally self-contained so the post can be moved
   without requiring a second report-specific stylesheet.
   ========================================================================== */

:root {
  --sage-bg: #0b1020;
  --sage-surface: #121a2b;
  --sage-surface-2: #172238;
  --sage-surface-3: #1d2940;
  --sage-border: rgba(148, 163, 184, 0.20);
  --sage-border-strong: rgba(96, 165, 250, 0.34);
  --sage-text: #e8edf7;
  --sage-text-soft: #c2ccdc;
  --sage-muted: #93a1b5;
  --sage-accent: #67b7ff;
  --sage-accent-strong: #8cc9ff;
  --sage-code: #0e1627;
  --sage-success: #79d6a3;
  --sage-warning: #f2c66d;
  --sage-shadow: 0 18px 50px rgba(0, 0, 0, 0.22);
}

/* Main report column ------------------------------------------------------ */

.post-content,
.post .post-content,
.page-content .post-content,
article.post .post-content {
  max-width: 980px;
  margin-inline: auto;
  color: var(--sage-text);
  line-height: 1.78;
  font-size: 17px;
  letter-spacing: 0.005em;
}

.post-content > p,
.post-content > ul,
.post-content > ol,
.post-content > blockquote {
  max-width: 900px;
}

.post-content p {
  margin: 0 0 1.15rem;
}

.post-content h2 {
  position: relative;
  margin: 3.6rem 0 1.15rem;
  padding-bottom: 0.55rem;
  color: #f8fafc;
  font-size: clamp(1.65rem, 2.4vw, 2.15rem);
  line-height: 1.2;
  letter-spacing: -0.025em;
  font-weight: 800;
}

.post-content h2::after {
  content: "";
  display: block;
  width: 58px;
  height: 3px;
  margin-top: 0.65rem;
  border-radius: 999px;
  background: linear-gradient(90deg, var(--sage-accent), transparent);
}

.post-content h3 {
  margin: 2.5rem 0 0.85rem;
  color: #f1f5fb;
  font-size: clamp(1.2rem, 1.7vw, 1.45rem);
  line-height: 1.35;
  font-weight: 750;
  letter-spacing: -0.012em;
}

.post-content strong {
  color: #f8fafc;
  font-weight: 750;
}

.post-content em {
  color: #d6deea;
}

.post-content a {
  color: var(--sage-accent);
  text-decoration: none;
  text-decoration-thickness: 1px;
  text-underline-offset: 3px;
  transition: color 160ms ease, text-decoration-color 160ms ease;
}

.post-content a:hover {
  color: var(--sage-accent-strong);
  text-decoration: underline;
}

/* Report metadata card ---------------------------------------------------- */

.meta-card {
  margin: 0 0 3.2rem;
  padding: 1.35rem 1.55rem;
  border: 1px solid var(--sage-border-strong);
  border-radius: 16px;
  background:
    radial-gradient(circle at 100% 0%, rgba(103, 183, 255, 0.10), transparent 35%),
    linear-gradient(145deg, rgba(23, 34, 56, 0.96), rgba(18, 26, 43, 0.96));
  box-shadow: var(--sage-shadow);
}

.meta-card p {
  margin: 0.55rem 0;
  display: grid;
  grid-template-columns: 135px minmax(0, 1fr);
  gap: 0.75rem;
  align-items: baseline;
}

.meta-card strong {
  color: #aebdd2;
  font-size: 0.88rem;
  text-transform: uppercase;
  letter-spacing: 0.075em;
}

.meta-card code {
  padding: 0.14rem 0.4rem;
  border: 1px solid rgba(148, 163, 184, 0.20);
  border-radius: 6px;
  background: rgba(15, 23, 42, 0.72);
  color: #d8e3f2;
  font-size: 0.84em;
}

/* Inline code ------------------------------------------------------------- */

.post-content :not(pre) > code {
  padding: 0.16em 0.42em;
  border: 1px solid rgba(148, 163, 184, 0.18);
  border-radius: 6px;
  background: rgba(15, 23, 42, 0.78);
  color: #dbeafe;
  font-size: 0.88em;
  white-space: break-spaces;
}

/* Code / architecture blocks ------------------------------------------------
   Keep architecture content intact, but make the visual treatment deliberate.
   -------------------------------------------------------------------------- */

.post-content pre {
  margin: 1.65rem 0 2rem;
  padding: 1.35rem 1.45rem;
  overflow-x: auto;
  border: 1px solid rgba(148, 163, 184, 0.18);
  border-radius: 14px;
  background:
    linear-gradient(145deg, rgba(18, 26, 43, 0.98), rgba(10, 16, 30, 0.98));
  box-shadow: 0 12px 34px rgba(0, 0, 0, 0.18);
  scrollbar-width: thin;
}

.post-content pre code {
  display: block;
  padding: 0;
  border: 0;
  background: transparent;
  color: #dbe5f2;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
               "Liberation Mono", "Courier New", monospace;
  font-size: 0.88rem;
  line-height: 1.62;
  white-space: pre;
}

/* Tables ------------------------------------------------------------------ */

.post-content table {
  width: 100%;
  margin: 1.55rem 0 2.1rem;
  border: 1px solid var(--sage-border);
  border-collapse: separate;
  border-spacing: 0;
  overflow: hidden;
  border-radius: 14px;
  background: var(--sage-surface);
  box-shadow: 0 10px 28px rgba(0, 0, 0, 0.15);
}

.post-content thead th {
  padding: 0.85rem 1rem;
  border-bottom: 1px solid var(--sage-border-strong);
  background: linear-gradient(180deg, #1c2a43, #18253a);
  color: #f5f8fc;
  font-size: 0.86rem;
  font-weight: 800;
  letter-spacing: 0.055em;
  text-transform: uppercase;
  text-align: left;
}

.post-content tbody td {
  padding: 0.78rem 1rem;
  border-bottom: 1px solid rgba(148, 163, 184, 0.11);
  color: #d8e1ee;
  background: #111a2a;
  vertical-align: top;
}

.post-content tbody tr:nth-child(even) td {
  background: #162137;
  color: #e0e8f4;
}

.post-content tbody tr:hover td {
  background: #1c2b45;
  color: #f8fafc;
}

.post-content tbody tr:last-child td {
  border-bottom: 0;
}

.post-content td a {
  font-weight: 650;
}

/* Numeric result columns stay visually crisp. */
.post-content th:last-child,
.post-content td:last-child {
  white-space: nowrap;
}

/* Figures and screenshots ------------------------------------------------- */

.post-content figure {
  margin: 2.25rem 0 2.5rem;
  padding: 0;
  text-align: center;
}

.post-content figure img {
  display: block;
  width: auto;
  max-width: 100%;
  height: auto;
  margin: 0 auto;
  border: 1px solid var(--sage-border);
  border-radius: 14px;
  background: #0b1020;
  box-shadow: 0 18px 45px rgba(0, 0, 0, 0.28);
}

.post-content figcaption {
  max-width: 820px;
  margin: 0.85rem auto 0;
  color: var(--sage-muted);
  font-size: 0.88rem;
  line-height: 1.55;
}

/* Blockquotes ------------------------------------------------------------- */

.post-content blockquote {
  margin: 2rem 0;
  padding: 1.25rem 1.45rem;
  border: 1px solid rgba(103, 183, 255, 0.22);
  border-left: 4px solid var(--sage-accent);
  border-radius: 0 13px 13px 0;
  background:
    linear-gradient(145deg, rgba(20, 38, 61, 0.88), rgba(15, 27, 45, 0.88));
  color: #d8e3f0;
  box-shadow: 0 10px 28px rgba(0, 0, 0, 0.12);
}

.post-content blockquote p:last-child {
  margin-bottom: 0;
}

/* Lists ------------------------------------------------------------------- */

.post-content ul,
.post-content ol {
  margin: 0.75rem 0 1.5rem;
  padding-left: 1.5rem;
}

.post-content li {
  margin: 0.38rem 0;
  padding-left: 0.15rem;
}

.post-content li::marker {
  color: var(--sage-accent);
  font-weight: 700;
}

/* Horizontal separators --------------------------------------------------- */

.post-content hr {
  margin: 3rem 0;
  border: 0;
  border-top: 1px solid var(--sage-border);
}

/* Optional site-header hooks ----------------------------------------------
   These cover common Jekyll layout class names without changing report
   content. If the theme uses different class names, the site-wide header
   stylesheet/layout must be updated there as well.
   -------------------------------------------------------------------------- */

header.site-header,
.site-header,
.masthead {
  min-height: 88px;
}

header.site-header .site-title,
.site-header .site-title,
.masthead .site-title {
  font-size: clamp(1.35rem, 2.3vw, 2rem) !important;
  font-weight: 850 !important;
  letter-spacing: -0.025em;
}

header.site-header .site-nav,
.site-header .site-nav,
.masthead .site-nav {
  font-size: 0.98rem;
  font-weight: 650;
}

header.site-header .site-nav a,
.site-header .site-nav a,
.masthead .site-nav a {
  transition: color 160ms ease, opacity 160ms ease;
}

/* Responsive layout ------------------------------------------------------- */

@media (max-width: 760px) {
  .post-content,
  .post .post-content,
  .page-content .post-content,
  article.post .post-content {
    font-size: 16px;
  }

  .meta-card {
    padding: 1rem 1.05rem;
    border-radius: 13px;
  }

  .meta-card p {
    display: block;
    margin: 0.85rem 0;
  }

  .meta-card strong {
    display: block;
    margin-bottom: 0.2rem;
  }

  .post-content table {
    display: block;
    overflow-x: auto;
    white-space: normal;
  }

  .post-content thead th,
  .post-content tbody td {
    min-width: 130px;
  }

  .post-content pre {
    padding: 1rem;
    border-radius: 11px;
  }

  .post-content pre code {
    font-size: 0.78rem;
  }

  .post-content h2 {
    margin-top: 2.8rem;
  }
}

@media (prefers-reduced-motion: reduce) {
  .post-content a,
  .site-header a,
  .masthead a {
    transition: none;
  }
}

/* Report-specific visual panels ----------------------------------------- */

.report-intro {
  margin-bottom: 3rem;
}

.report-intro::before {
  content: "GSoC 2026 · Final Report";
  display: block;
  margin: 0 0 0.55rem;
  color: var(--sage-accent);
  font-size: 0.78rem;
  font-weight: 800;
  letter-spacing: 0.13em;
  text-transform: uppercase;
}

.report-intro .meta-card {
  margin-bottom: 0;
}

.architecture-panel,
.source-tree-panel,
.process-panel {
  margin: 1.5rem 0 2.4rem;
}

.architecture-panel pre {
  border-color: rgba(103, 183, 255, 0.30);
  background:
    radial-gradient(circle at 0% 0%, rgba(103, 183, 255, 0.11), transparent 36%),
    linear-gradient(145deg, #101b2f, #0b1221);
}

.architecture-panel pre code {
  color: #e6edf7;
  font-size: 0.86rem;
}

.source-tree-panel pre {
  background:
    linear-gradient(145deg, #111a2a, #0b1220);
}

.process-panel pre {
  border-color: rgba(121, 214, 163, 0.24);
  background:
    radial-gradient(circle at 100% 0%, rgba(121, 214, 163, 0.08), transparent 35%),
    linear-gradient(145deg, #111b2b, #0b1322);
}

.process-panel pre code {
  color: #e1ece8;
}

/* If the site's shared layout exposes one of these common header hooks,
   the report can improve its header without modifying the site template. */
body > header.site-header,
body > .site-header,
body > .masthead,
header.site-header {
  border-bottom: 1px solid rgba(148, 163, 184, 0.16);
  background:
    linear-gradient(180deg, rgba(11, 16, 32, 0.98), rgba(11, 16, 32, 0.90));
  backdrop-filter: blur(14px);
}

body > header.site-header .site-title,
body > .site-header .site-title,
body > .masthead .site-title,
header.site-header .site-title {
  font-size: clamp(1.55rem, 2.8vw, 2.15rem) !important;
  font-weight: 850 !important;
  letter-spacing: -0.035em;
  line-height: 1.05;
}

body > header.site-header .site-nav a,
body > .site-header .site-nav a,
body > .masthead .site-nav a,
header.site-header .site-nav a {
  color: #b8c5d8;
  font-weight: 650;
}

body > header.site-header .site-nav a:hover,
body > .site-header .site-nav a:hover,
body > .masthead .site-nav a:hover,
header.site-header .site-nav a:hover {
  color: #f4f8ff;
}

</style>

<div class="report-intro">
<div class="meta-card" markdown="1">

Organization: PEcAn Project

Mentors: David LeBauer @dlebauer, Pratik Pakhale @pratikpakhale, Nihar Sanda @koolgax99

Student: Abhishek Kumar Rai @Abhishek-Kumar-Rai5

Repository: github.com/PecanProject/sage

Pull requests: PecanProject/sage/pulls · author

</div>
</div>

Project Summary

Calibrating and validating a crop model against published field-trial data starts with a slow, manual step: a

curator reads a paper and copies sites, treatments, methods and measured values into a spreadsheet, by hand, one

paper at a time. SAGE automates that first pass. Given a PDF, it produces a structured intermediate

representation (IR) — citations, studies, sites, species, crops, treatments, methods, variables, management events,

coverage and observations — in the shape PEcAn's Calibration and Validation Data Collection Protocol defines, with

every value tied to the exact source text it came from. Nothing is guessed: a value the paper does not state stays

explicitly UNRESOLVED with a reason, and a scientist reviews every record against the live PDF in a web UI

before it is accepted.

The system was built from an empty repository this program: schema, storage, a FastAPI tool service, two sealed LLM

agents, a deterministic orchestrator, a Streamlit review UI, and a from-scratch table-reconstruction engine — 190

files, ~57,000 lines, covered by 1,138 passing tests, evaluated end-to-end on a real, previously-unseen paper.

Contributions at a Glance

All PRs are against the upstream organization repository, PecanProject/sage. An earlier pass at the first six

milestones was closed and rebuilt from a clean foundation once the target architecture was settled — this table is

that current, open stack.

| # | PR | What it added |

|---|---|---|

| 1 | #7 | Project foundation |

| 2 | #14 | IR schema & storage |

| 3 | #15 | Deterministic validation & document reading |

| 4 | #16 | Agent service & tool interface |

| 5 | #17 | Extraction orchestration & enumeration |

| 6 | #18 | Streamlit data & app infrastructure |

| 7 | #19 | Review interface |

| 8 | #20 | Document processing adapter (docproc) |

| 9 | #21 | Pipeline hardening from real-paper evaluation |

| 10 | #22 | Review table redesign + DOI feature request |

| 11 | #23 | This report |

The Problem

The protocol this project targets (D. LeBauer's working draft) exists because PEcAn's calibration and validation

pipeline needs structured, source-traceable observations, and today those are curated by hand from PDFs. Two

properties of that manual process are hard to give up when automating it: every number must be traceable to the

sentence or table cell it came from, and "the paper doesn't say" must remain a legitimate, recorded answer rather

than being silently invented. SAGE's design follows directly from those two constraints — grounding and honesty

about missing information are enforced in code, not left to prompt instructions an LLM might drift away from.

Architecture

<div class="architecture-panel">

PDF ──Marker──▶ docproc adapter ──▶ content.md + provenance.json   (text blocks with ⟦b:NNNN⟧ anchors)

                                          │

                     ┌────────────────────┴─────────────────────┐

                     ▼                                          ▼

             ir_service (FastAPI, 15 endpoints)       opencode agents

             schema · validation · storage            extractor  (sees the paper, never the schema)

                     ▲                                converter  (sees the schema, never the paper)

                     │                                ir_validator (observe-only second opinion)

                     └──────────── orchestrator ◀─────────────────┘

                     enumerate → extract → grounding gate → convert → validate → commit

                                          │

                                          ▼

                       results store ──▶ Streamlit review UI ──▶ corrections log


The extractor and the converter are kept apart on purpose: the model that reads the paper never sees the IR schema,

and the model that fills the schema never sees the paper, so a value has to be quoted from the source and pass a

deterministic text-match before it can enter a record. Neither model is ever allowed to call propose_record,

commit_record or flag_unresolved directly — the orchestrator is the only caller of all three, so a record can

never be committed on an LLM's own say-so.

</div>

<div class="source-tree-panel">

src/

  docproc/            Marker output → content.md + provenance.json adapter, QC gate, batch runner

  pipeline/           orchestrator, ir_service (FastAPI), IR + raw schemas, validators, content reader,

                       results / run / corrections stores, table pooling evidence, run configuration & locks

  opencode-config/     agent definitions (extractor, converter, ir_validator), skills, TypeScript tool plugin

  tests/               42 test modules + fixtures (trimmed paper text only)

streamlit_app/         review UI (paper library, PDF pane, record review table) + UI tests


The Work, PR by PR

1 · Foundation & IR schema (#7, #14)

12 Pydantic entity types — Citation, Study, Site, Species, Crop, Method, Treatment, TreatmentPair, Variable,

Management, Observation, Coverage. Every extracted field is wrapped in an ExtractedField[T] carrying a provenance

label — EXTRACTED, INFERRED (must state its inference basis), or UNRESOLVED (must state why, value forced to

null) — plus its source locator(s). Records live in an append-only ir-store/, and a schema fingerprint check

refuses to serve a request if the running validation service has a stale copy of the code in memory.

2 · Deterministic validation & document reading (#15)

Construction-time invariants (e.g. reported_effect_scope ↔ aggregated_over_factors coupling, Management.date/

amount may never be INFERRED) are enforced by Pydantic model validators; whole-graph invariants (duplicate ids,

dangling references, one control treatment per study/site, Treatment-name uniqueness) are enforced separately

against the full dataset. This is where the grounding gate lives: every quoted excerpt must literally appear in

its cited source block — an agent's claim that a value is supported is never trusted on its own.

3 · Agent service & tool interface (#16)

A FastAPI app, the single writer of every record: propose_record (dry-run validate), commit_record,

flag_unresolved, get_schema, lookup_vocab, apply_reconstruction (date/stat/factorial-expansion helpers), and

read-only tools (read_section, read_table, read_table_row/cell, read_nearby, list_sections, …) that let an

agent cite exactly the block it read. Deterministic validation re-runs on every proposal and every commit. A

per-record attempt counter forces a hard turn cap (4 attempts) before a record must be flagged unresolved instead of

retried forever.

4 · Extraction orchestration & enumeration (#17)

The one place a model is ever invoked, and the only place the outer control flow lives — no agent decides what

happens after it answers. Per entity type it enumerates candidates, then runs each candidate through

Extraction (sealed to the paper) → grounding gate → Conversion (sealed to the schema) → deterministic validation →

readiness classification (ready / unresolved / error) → one bounded AI-validator correction pass → commit.

Entity types are processed in dependency order (Citation/Site/Species first, Observation last), and a multi-record

entity's candidates are linked to the specific prerequisite record their own cited text names — never guessed onto

one of several plausible ones.

5 · Streamlit infrastructure & review interface (#18, #19)

A two-pane workstation: the real PDF on the left, extracted records on the right, one row per field as

result | source | actions.

<figure>

  <img src="/assets/images/screenshot-library.png" alt="Streamlit paper library page">
  <img src="/assets/images/screenshot-library_panel.png" alt="Streamlit paper library page">
  <img src="/assets/images/screenshot-library_processing.png" alt="Streamlit paper library page">

  <figcaption>The paper library — upload, run Marker processing, and see each paper's processed / extracted / error status.</figcaption>

</figure>

<figure>

  <img src="/assets/images/screenshot-workspace.png" alt="Two-pane review workspace">

  <figcaption>The two-pane review workspace: the source PDF on the left, extracted records on the right.</figcaption>

</figure>

The source column shows the quoted evidence with its page and section; an eye icon jumps the PDF pane to the exact

highlighted polygon, resolved from real Marker geometry — never from an LLM-reported page number, which was

confirmed unreliable.

<figure>

  <img src="/assets/images/screenshot-field-review.png" alt="Expanded record showing result, source and action columns">

  <figcaption>One expanded record: each field as a result | source | actions row.</figcaption>

</figure>

<figure>

  <img src="/assets/images/screenshot-pdf-evidence.png" alt="PDF pane highlighting the cited source block">

  <figcaption>Clicking the eye icon jumps the PDF pane straight to the cited block.</figcaption>

</figure>

Actions are approve, edit (correct value / add note / relocate evidence / tell the agent), all appended to an

immutable corrections log — the original extraction is never overwritten. Link fields (*_id) are hidden from

review entirely, and Citation (auto-filled, low-priority bibliographic metadata) is collapsed by default.

6 · Document processing adapter — docproc (#20)

Turns Marker's raw block-tree JSON into content.md — one stable anchor after every rendered block — and

provenance.json, mapping each anchor to its page, section path, and polygon. Handles real Marker quirks discovered

against live papers: double-escaped <sup> corruption from --use_llm, geometric row/column clustering for table

cells, page-split table continuations, unhandled block types logged rather than dropped. A three-check QC gate

(round-trip anchor consistency, tag-leak sweep, block-type coverage) flags a bad conversion before extraction ever

starts on it.

7 · Pipeline hardening from real-paper evaluation (#21)

Everything in §Key Innovation below, plus: safer Method

matching (it now uses only the variable and method hint, so a Treatment level such as "Fallow" can no longer leak

into and win a Method match), a null/blank source fact dropped as ungrounded auxiliary evidence instead of failing

the whole record it belongs to, provider-failure classification, and the 1,046-test backend suite.

Provider resilience. Empty, timed-out, or malformed model responses (a confirmed real gpt-oss-120b/vLLM "harmony"

tool-call corruption) are classified separately from genuine content failures and spend a separate cooldown budget,

not one of a record's numbered content-retry attempts, so a transient provider burst never costs a record its real

chance to converge.

Run infrastructure. A per-paper file lock refuses to let two runs interleave on the same paper. Every run's

manifest records its exact model/provider, git commit + dirty-diff hash, and a fingerprint of the schema/validators/

prompts/agent configs actually used, so any run's output can be traced back to precisely the code that produced it.

8 · Review table redesign (#22)

The result | source | actions layout above replaced an earlier, code-heavy field view (EXT/UNR/REF status

codes gone — state is written in words, and only when it isn't the ordinary case) — direct mentor feedback from

review of the earlier UI. Also added: a feature request to fill Citation metadata from a DOI via Crossref instead of

relying on the PDF's own text, since the validation paper's DOI is nowhere in its PDF.

Key Innovation: Deterministic Table Reconstruction

The single largest piece of original engineering this program. Free-form LLM enumeration was confirmed, against a

real paper (a table crossing 6 populations × 3 maturities × 2 sites × 4 measures — up to 144 real reported values),

to collapse the whole table into one candidate per measure, in two independent runs, regardless of prompt tuning —

the model simply cannot reliably hold that much combinatorial structure in one free-form answer. Rather than

continuing to prompt-engineer around it, the "how many distinct values exist" arithmetic was moved into

deterministic code:

<div class="process-panel">

Step A — discover every Table block in the document (pure code, from provenance.json)

Step B — one narrow LLM call per table: classify its role, its factor dimensions

         (treatment / crop / time / site / variable) and how each is encoded

         (rows / columns / context), plus any factor the table's values are

         pooled/averaged over — bounded retry, plus a deterministic

         reconstruction sanity check against the table's own raw cell text

Step C — pure deterministic cross-product of Step B's structure into one

         candidate per real, individually-reported cell

Step D — merge with free-form enumeration, told which anchors are already

         covered so it does not re-report (and duplicate) them


Everything downstream of candidate generation (linking, extraction, conversion, provenance validation, the

refuse-to-guess gate, AI validation) is unchanged — this mechanism only changes what populates the candidate list.

Treatment identity for a table-sourced candidate is computed from canonical semantic content (normalized factor

levels + resolved site), not from the table's own raw key names — a real early run produced 131 Treatment records

where only ~68 were genuinely distinct, because "Location" vs. "Site" and "Ames, IA" vs. "Ames" were treated as

different keys.

Testing & QA

All tests run without a network connection or a live model: model calls are replaced by a canned sequence of

scripted responses so control flow (retry counts, when flag_unresolved fires, that AI validation stays

observe-only, that nothing commits until deterministic validation passes) is tested fast and deterministically; the

IR service itself runs for real, since deterministic validation is the hard gate the whole pipeline depends on.

Every test file is written against a real, cited failure (a specific run id and paper), not a synthetic scenario

invented in the abstract.

src/tests/                 42 modules  →  1046 passed

streamlit_app/tests/        8 modules  →    87 passed, 13 skipped (need a real browser)

src/docproc/test_marker_adapter.py     →     5 passed


Coverage: orchestrator control flow and attempt budgets, the grounding gate, IR schema and whole-graph validators,

the IR service, table reconstruction (roles, factors, continuation, pooling, method hints, variable declarations),

temporal context, canonical treatment identity, provider failures and resilience, run isolation and locks, results/

corrections stores, review-table rendering, field-state wording, approve/edit actions, link-field hiding, PDF-pane

wiring. (All three counts re-run and confirmed at report time.)

Real-World Validation

One complete, unmodified end-to-end pipeline execution against a real, previously-unprocessed paper —

Barrios-Masias, Cantwell & Jackson, 2010, "Cultivar mixtures of processing tomato in an organic agroecosystem" —

model gpt-oss-120b, no code changed mid-run:

| Measurement | Result |

|---|---:|

| Runtime | 3 h 24 min |

| Model calls | 481 (extractor 201 · converter 174 · ir-validator 106) |

| Records produced | 133 |

| Ready | 96 |

| Unresolved | 35 |

| Error | 2 |

| Provider-failed rounds | 47 (recovered without costing a content-retry attempt, bar one) |

| Ready observations | 13 / 42 |

Every one of the 13 ready Observation values was checked by hand against the paper text and is correct. The run also

surfaced concrete, still-open problems rather than hiding them behind a "ready" status:

Method pool incomplete for this run — no soil-sampling, chamber-gas, or plant-sampling Method was extracted, so

  8 soil observations and the 16 non-PAR cells of Table 1 were correctly refused (left unresolved) rather than

  linked to a wrong Method.

Two wrong Method links slipped through anyway (fruit phosphorus → pH meter; harvestable fruit → colour

  reflectance) — evidence that Method matching needs a stronger check, not just a broader pool.

Date confusions in Management — a cover-crop incorporation date and a harvest date were attached to the wrong

  events (weeding, sulfur application).

Observation timing (days-after-planting) is not yet carried through into temporal_info, and Treatments are not

  yet linked to their Study.

Figures are raster images in the source PDF, so nothing is extracted from them; the paper's DOI is not printed

  anywhere in the PDF text.

Technologies Used

Python · Pydantic · FastAPI + uvicorn (IR service) · httpx · pytest (1,138 tests, all offline/mocked) ·

opencode LLM agent runtime (extractor / converter / ir-validator, each sealed to different tools) ·

Streamlit + pypdfium2 (PDF-anchored review UI) · Marker (PDF → structured block tree) · pandas / numpy.

Known Limitations & Next Steps

Stabilize Method enumeration (it drives most of the unresolved Observations) and re-check Method matching for

   short/ambiguous names.

Fix Management date parsing so an event's date is never attached to a different event.

Carry days-after-planting into Observation.temporal_info, and link Treatments to their Study.

Represent factorial designs (e.g. cover crop × cultivar mixture) more completely at the candidate-enumeration

   level.

Citation metadata from DOI/Crossref instead of relying on the PDF's own text.

Shorten the provider cool-down schedule — at the 2-hour mark of the validation run above, roughly half of elapsed

   wall-clock time was spent waiting out provider cool-downs rather than making forward progress.

Expand evaluation beyond one paper to a scored gold dataset and the remaining protocol validation papers.

A Note From Me

The part of this project I'd point to first isn't any one entity type or endpoint — it's the number of places the

pipeline is designed to say "I don't know" instead of guessing. Watching free-form enumeration silently collapse a

144-value table down to four candidates was the moment that stopped feeling like a prompting problem and started

feeling like an architecture problem — the fix wasn't a better prompt, it was moving the counting into code the

model never touches. Most of what's in this report is that same move, repeated: wherever an LLM's own judgment

couldn't be trusted to be conservative on its own, something deterministic sits in front of it and says no.



Thanks to David, Pratik and Nihar for the direction throughout — the review-table redesign in particular came

straight out of their feedback on the earlier version.

Acknowledgments

Thanks to David LeBauer, Pratik Pakhale and Nihar Sanda for mentoring this project, and to the PEcAn community for

the Calibration and Validation Data Collection Protocol this pipeline was built to serve.