---
layout: post
title: "SAGE: LLM extraction of agronomic papers into a structured, reviewable IR"
date: 2026-09-22 00:00:00 +0000
excerpt: >-
  Google Summer of Code 2026 final report — building SAGE, an LLM pipeline that turns published crop-science
  papers into a structured, source-grounded, human-reviewed intermediate representation for the PEcAn project.
---

Google Summer of Code 2026, working with the [PEcAn Project](https://github.com/PecanProject/sage), mentored by
[David LeBauer](https://github.com/dlebauer), [Pratik Pakhale](https://github.com/pratikpakhale) and
[Nihar Sanda](https://github.com/koolgax99). Code and open pull requests:
[PecanProject/sage/pulls](https://github.com/PecanProject/sage/pulls?q=is%3Apr+author%3AAbhishek-Kumar-Rai5).

## Abstract

Calibrating and validating a crop model against published field-trial data starts with a slow, manual step: a curator
reads a paper and copies sites, treatments, methods and measured values into a spreadsheet, by hand, one paper at a
time. SAGE automates that first pass. Given a PDF, it produces a structured intermediate representation (IR) —
citations, studies, sites, species, crops, treatments, methods, variables, management events, coverage and
observations — in the shape the PEcAn *Calibration and Validation Data Collection Protocol* defines, with every value
tied to the exact source text it came from. Nothing is guessed: a value the paper does not state stays explicitly
**UNRESOLVED** with a reason, and a scientist reviews every record against the live PDF in a web UI before it is
accepted. The system was built from an empty repository this program (schema, storage, a FastAPI tool service, two
sealed LLM agents, a deterministic orchestrator, a Streamlit review UI, and a from-scratch table-reconstruction
engine — 190 files, ~57k lines), is covered by 1,138 passing tests, and was evaluated end-to-end on a real,
previously-unseen paper.

## 1. Motivation

The protocol this project targets (D. LeBauer's working draft) exists because PEcAn's calibration and validation
pipeline needs structured, source-traceable observations, and today those are curated by hand from PDFs. Two
properties of that manual process are hard to give up when automating it: every number must be traceable to the
sentence or table cell it came from, and "the paper doesn't say" must remain a legitimate, recorded answer rather
than being silently invented. SAGE's design follows directly from those two constraints — grounding and honesty
about missing information are enforced in code (Pydantic validators, a text-grounding gate, a review UI), not left to
prompt instructions an LLM might drift away from.

## 2. What SAGE does

```
PDF ──Marker──▶ docproc adapter ──▶ content.md + provenance.json   (text blocks with ⟦b:NNNN⟧ anchors)
                                          │
                     ┌────────────────────┴─────────────────────┐
                     ▼                                          ▼
             ir_service (FastAPI, 15 endpoints)       opencode agents
             schema · validation · storage            extractor  (sees the paper, never the schema)
                     ▲                                converter  (sees the schema, never the paper)
                     │                                ir_validator (observe-only second opinion)
                     └──────────── orchestrator ◀─────────────────┘
                     enumerate → extract → grounding gate → convert → validate → commit
                                          │
                                          ▼
                       results store ──▶ Streamlit review UI ──▶ corrections log
```

The extractor and the converter are kept apart on purpose: the model that reads the paper never sees the IR schema,
and the model that fills the schema never sees the paper, so a value has to be quoted from the source and pass a
deterministic text-match before it can enter a record. Neither model is ever allowed to call `propose_record`,
`commit_record` or `flag_unresolved` directly — the orchestrator is the only caller of all three, so a record can
never be committed on an LLM's own say-so.

## 3. Repository layout

```
src/
  docproc/            Marker output → content.md + provenance.json adapter, QC gate, batch runner
  pipeline/           orchestrator, ir_service (FastAPI), IR + raw schemas, validators, content reader,
                       results / run / corrections stores, table pooling evidence, run configuration & locks
  opencode-config/     agent definitions (extractor, converter, ir_validator), skills, TypeScript tool plugin
  tests/               42 test modules + fixtures (trimmed paper text only)
streamlit_app/         review UI (paper library, PDF pane, record review table) + UI tests
docs/                  reports, feature requests
```

## 4. Main components

- **Document processing (`src/docproc/`).** Turns Marker's raw block-tree JSON into `content.md` — one stable anchor
  after every rendered block — and `provenance.json`, mapping each anchor to its page, section path, and polygon, so
  any extracted value can be shown on the actual PDF page later. Handles real Marker quirks discovered against live
  papers (double-escaped `<sup>` corruption from `--use_llm`, geometric row/column clustering for table cells,
  page-split table continuations, unhandled block types logged rather than dropped). A three-check QC gate
  (round-trip anchor consistency, tag-leak sweep, block-type coverage) flags a bad conversion before extraction ever
  starts on it.
- **IR schema (`pipeline/ir_schema.py`).** 12 Pydantic entity types (Citation, Study, Site, Species, Crop, Method,
  Treatment, TreatmentPair, Variable, Management, Observation, Coverage). Every extracted field is wrapped in an
  `ExtractedField[T]` carrying a provenance label — `EXTRACTED`, `INFERRED` (must state its inference basis), or
  `UNRESOLVED` (must state why, value forced to null) — plus its source locator(s). Construction-time invariants
  (e.g. `reported_effect_scope` ↔ `aggregated_over_factors` coupling, `Management.date`/`amount` may never be
  `INFERRED`) are enforced by Pydantic model validators; whole-graph invariants (duplicate ids, dangling references,
  one control treatment per study/site, Treatment-name uniqueness) are enforced separately in `validators.py` against
  the full dataset.
- **IR service (`ir_service.py`).** A FastAPI app, the single writer of every record: `propose_record` (dry-run
  validate), `commit_record`, `flag_unresolved`, `get_schema`, `lookup_vocab`, `apply_reconstruction` (date/stat/
  factorial-expansion helpers), and read-only tools (`read_section`, `read_table`, `read_table_row/cell`,
  `read_nearby`, `list_sections`, …) that let an agent cite exactly the block it read. Deterministic validation is
  re-run on every proposal and every commit — an agent's own claim that a record is valid is never trusted. A
  per-record attempt counter forces a hard turn cap (4 attempts) before a record must be flagged unresolved instead
  of retried forever, and a schema fingerprint check refuses to serve a request if the running process has a stale
  copy of the validation code in memory.
- **Extraction orchestrator (`pipeline/orchestrator.py`, ~6,300 lines).** The one place a model is ever invoked, and
  the only place the outer control flow lives — no agent decides what happens after it answers. Per entity type it
  enumerates candidates (free-form prose reading, plus a deterministic table-reconstruction pipeline for dense data
  tables — see §5), then runs each candidate through Extraction (sealed to the paper) → a **raw-evidence grounding
  gate** (every quoted excerpt must literally appear in its cited block) → Conversion (sealed to the schema) →
  deterministic validation → readiness classification (`ready` / `unresolved` / `error`) → one bounded AI-validator
  correction pass → commit. Entity types are processed in dependency order (Citation/Site/Species first, Observation
  last), and a multi-record entity's candidates are linked to the *specific* prerequisite record their own cited text
  names — a candidate is never guessed onto one of several plausible prerequisites.
- **Provider resilience.** Empty, timed-out, or malformed model responses (a confirmed real gpt-oss-120b/vLLM
  "harmony" tool-call corruption) are classified separately from genuine content failures and spend a separate
  cooldown budget, not one of a record's numbered content-retry attempts, so a transient provider burst never costs a
  record its real chance to converge. Repeated exhausted budgets drop the run to one round per record so a genuine
  outage costs minutes, not hours.
- **Run infrastructure.** A per-paper file lock (`run_lock.py`) refuses to let two runs interleave on the same paper.
  Every run's manifest records its exact model/provider, git commit + dirty-diff hash, and a fingerprint of the
  schema/validators/prompts/agent configs actually used, so any run's output can be traced back to precisely the code
  that produced it. Results are written per-run (`results/<paper>/<run_id>/`) with an atomic `LATEST` pointer, so a
  crashed or still-running run never corrupts what a reader sees.
- **Review UI (`streamlit_app/`).** A two-pane workstation: the real PDF on the left, extracted records on the right,
  one row per field as `result | source | actions`. The source column shows the quoted evidence with its page and
  section; an eye icon jumps the PDF pane to the exact highlighted polygon (resolved from `provenance.json`'s real
  Marker geometry, never from an LLM-reported page number, which was confirmed unreliable). Actions are approve, edit
  (correct value / add note / relocate evidence / tell the agent), all appended to an immutable corrections log —
  the original extraction is never overwritten. Link fields (`*_id`) are hidden from review entirely, and Citation
  (auto-filled, low-priority bibliographic metadata) is collapsed by default.

## 5. Table reconstruction (Steps A–D)

The single largest algorithmic piece of original engineering this program: free-form LLM enumeration was confirmed,
against a real paper (Daren-1997-Canopy, a table crossing 6 populations × 3 maturities × 2 sites × 4 measures — up to
144 real reported values), to collapse the whole table into one candidate per measure, in two independent runs,
regardless of prompt tuning — the model simply cannot reliably hold that much combinatorial structure in one
free-form answer. Rather than continuing to prompt-engineer around it, the "how many distinct values exist"
arithmetic was moved into deterministic code:

```
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
```

Everything downstream of candidate generation (linking, extraction, conversion, provenance validation, the
refuse-to-guess gate, AI validation) is unchanged — this mechanism only changes what populates the candidate list.
Treatment identity for a table-sourced candidate is computed from canonical semantic content (normalized factor
levels + resolved site), not from the table's own raw key names — a real early run produced 131 Treatment records
where only ~68 were genuinely distinct, because "Location" vs. "Site" and "Ames, IA" vs. "Ames" were treated as
different keys.

## 6. Prompt and rule changes

Agent system prompts (`extractor`, `converter`, `ir_validator`) are unchanged. What the orchestrator builds and feeds
them changed as follows:

| Prompt | Change |
|---|---|
| Table classification | Declares every table variable, including variables encoded as **row labels**, not just column headers; a table whose row levels are undeclared is retried once with feedback and otherwise accepted and flagged for review. The paper's Methods blocks are supplied verbatim so a method hint can be tied to real prose — a hint the prose does not support is withheld. Page-split table continuations are detected and folded into one logical table. |
| Enumeration | Names the exact list-field spelling for links (`treatment_ids`, plural) — fixed a bug where a singular example key made every proposed link silently vanish. Links a Management event to a condition only when the event's own cited text names it. Can ask the model to declare its own candidates' factor dimensions for comparison against table-sourced candidates. |
| Conversion | Tells the converter not to mark a field UNRESOLVED for a missing page number (the pipeline fills it deterministically). Passes a sealed, orchestrator-authored `CANDIDATE_CONTEXT` for treatment links instead of letting the model guess them. |
| Extraction, AI validation | Unchanged. |

Non-prompt rule changes: Method matching now uses only the variable and method hint, so a Treatment level such as
`"Fallow"` can no longer leak into (and win) a Method match. A `null`/blank source fact with no text is dropped as
ungrounded auxiliary evidence instead of failing the whole record it belongs to.

## 7. Testing

All tests run without a network connection or a live model: model calls are replaced by a canned sequence of scripted
responses so control flow (retry counts, when `flag_unresolved` fires, that AI validation stays observe-only, that
nothing commits until deterministic validation passes) is tested fast and deterministically; the IR service itself
runs for real, since deterministic validation is the hard gate the whole pipeline depends on. Every test file is
written against a real, cited failure (a specific run id and paper), not a synthetic scenario invented in the
abstract.

| Suite | Result | Scope |
|---|---|---|
| Backend (42 modules) | **1046 passed** | orchestrator control flow and attempt budgets, the grounding gate, IR schema and whole-graph validators, IR service, table reconstruction (roles, factors, continuation, pooling, method hints, variable declarations), temporal context, canonical treatment identity, provider failures and resilience, run isolation and locks, results/corrections stores |
| Streamlit UI (8 modules) | **87 passed, 13 skipped** | review-table rendering (headless), field-state wording, approve/edit actions, link-field hiding, PDF-pane wiring; the 13 skips need a real browser |
| Document adapter | **5 passed** | anchor and provenance generation against real Marker JSON |

*(All three counts re-run and confirmed at report time.)*

## 8. Real-paper validation

One complete, unmodified end-to-end pipeline execution against a real, previously-unprocessed paper —
*Barrios-Masias, Cantwell & Jackson, 2010, "Cultivar mixtures of processing tomato in an organic agroecosystem"* —
model `gpt-oss-120b`, no code changed mid-run:

| | |
|---|---|
| Runtime | 3 h 24 min |
| Model calls | 481 (extractor 201 · converter 174 · ir-validator 106) |
| Records produced | 133 — **96 ready**, 35 unresolved, 2 error |
| Provider-failed rounds | 47 (1 loop hit its terminal budget; the rest recovered without costing a content-retry attempt) |
| Citation | ready (no longer blocks the rest of the paper, as it once did) |
| Observations | 42 total — **13 ready**, 28 unresolved, 1 error |

Every one of the 13 ready Observation values was checked by hand against the paper text and is correct. The run also
surfaced concrete, still-open problems rather than hiding them behind a "ready" status:

- **Method pool incomplete for this run** — no soil-sampling, chamber-gas, or plant-sampling Method was extracted, so
  8 soil observations and the 16 non-PAR cells of Table 1 were correctly refused (left unresolved) rather than linked
  to a wrong Method.
- **Two wrong Method links** slipped through anyway (fruit phosphorus → pH meter; harvestable fruit → colour
  reflectance) — evidence that Method matching needs a stronger check, not just a broader pool.
- **Date confusions in Management** — a cover-crop incorporation date and a harvest date were attached to the wrong
  events (weeding, sulfur application).
- Observation timing (days-after-planting) is not yet carried through into `temporal_info`, and Treatments are not
  yet linked to their Study (`Treatment.study_id` stays UNRESOLVED by design pending that reconciliation step).
- Figures are raster images in the source PDF, so nothing is extracted from them; the paper's DOI is not printed
  anywhere in the PDF text (motivating the Crossref feature request below).

## 9. Known limitations and next steps

1. Stabilize Method enumeration (it drives most of the unresolved Observations) and re-check Method matching for
   short/ambiguous names.
2. Fix Management date parsing so an event's date is never attached to a different event.
3. Carry days-after-planting into `Observation.temporal_info`, and link Treatments to their Study.
4. Represent factorial designs (e.g. cover crop × cultivar mixture) more completely at the candidate-enumeration
   level.
5. Citation metadata from DOI/Crossref instead of relying on the PDF's own text (feature request below).
6. Shorten the provider cool-down schedule — at the 2-hour mark of the validation run above, roughly half of elapsed
   wall-clock time was spent waiting out provider cool-downs rather than making forward progress.
7. Expand evaluation beyond one paper to a scored gold dataset and the remaining protocol validation papers.

## 10. Reference documents

- **Calibration and Validation Data Collection Protocol** (D. LeBauer, working draft) — the target the IR follows,
  with its schema (`datapackage.json`): internal reference material used to design the IR, not itself part of the
  delivered code.
- **Feature request — fill Citation metadata from a DOI (habanero / Crossref) instead of the PDF text:**
  [view the proposal](https://github.com/PecanProject/sage/pull/22/files).

## 11. Code and pull requests

Built from an empty repository this program: 190 files, ~57,200 lines added, across the phases below. All PRs are
against the upstream organization repository, `PecanProject/sage`; an earlier pass at the first six milestones was
closed and rebuilt from a clean foundation once the target architecture was settled — the PRs below are that
current, open stack.

| Phase | What it added | PR |
|---|---|---|
| Foundation | `.gitignore`, `requirements.txt` written against the target architecture | [#7](https://github.com/PecanProject/sage/pull/7) |
| IR schema & storage | `ir_schema.py` (12 entity types), append-only `ir-store/`, schema fingerprinting | [#14](https://github.com/PecanProject/sage/pull/14) |
| Deterministic validation & document reading | whole-graph `validators.py` (Table 19), `content_reader.py` | [#15](https://github.com/PecanProject/sage/pull/15) |
| Agent service & tool interface | `ir_service.py` (FastAPI), extractor/converter/ir-validator agent configs, TS tool plugin | [#16](https://github.com/PecanProject/sage/pull/16) |
| Extraction orchestration & enumeration | `orchestrator.py` core control loop, candidate enumeration, run store/config/lock | [#17](https://github.com/PecanProject/sage/pull/17) |
| Streamlit data & app infrastructure | `api_client.py`, `sage_paths.py`, `state.py`, wiring real data in place of mocks | [#18](https://github.com/PecanProject/sage/pull/18) |
| Review interface | two-pane workspace, PDF pane, `provenance_adapter.py`, record/field components | [#19](https://github.com/PecanProject/sage/pull/19) |
| Deterministic table enumeration + safety layer | Steps A–D table reconstruction (§5), raw-evidence grounding gate, candidate-collision detection | folded into the pipeline |
| Document processing adapter (docproc) | `marker_adapter.py`, `qc_gate.py`, `prepare_papers.py`, batch QC runner | [#20](https://github.com/PecanProject/sage/pull/20) |
| Pipeline hardening | grounding/null-fact fixes, pooling evidence, safer Method matching, provider resilience, run isolation, the 1,046-test backend suite | [#21](https://github.com/PecanProject/sage/pull/21) |
| Review table redesign | `result \| source \| actions` layout, link-field hiding, Citation collapsed, DOI/Crossref feature request | [#22](https://github.com/PecanProject/sage/pull/22) |
| Final report | this write-up | [#23](https://github.com/PecanProject/sage/pull/23) |

Excluded on purpose, everywhere: source PDFs, raw Marker JSON, and generated results/run logs — none of that is
committed code.

## Acknowledgments

Thanks to David LeBauer, Pratik Pakhale and Nihar Sanda for mentoring this project, and to the PEcAn community for
the Calibration and Validation Data Collection Protocol this pipeline was built to serve.
