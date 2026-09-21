---
Title: "SAGE — GSoC 2026 Final Report"
---

# LLM-Assisted Extraction of Agronomic and Ecological Experiments into Structured Data

**Google Summer of Code 2026 · PEcAn Project · Contributor: [Abhishek Kumar Rai](https://github.com/Abhishek-Kumar-Rai5)
**

**Mentors:** David LeBauer, Nihar Sanda, Pratik Pakhale

**Code and PRs:** https://github.com/PecanProject/sage/pulls

## 1. What SAGE does

SAGE reads a published crop-science paper and produces a structured intermediate representation (IR) of the information reported in the paper: sites, species, crops, treatments, methods, variables, management events, coverage and observations.

The IR follows the Calibration and Validation Data Collection Protocol. Every extracted value carries source provenance; information not stated clearly in the paper remains **UNRESOLVED**; and the result is presented for human review before acceptance.

```text
PDF
 │
 ▼
Marker
 │
 ▼
docproc
 │
 ├── content.md
 └── provenance.json
 │
 ▼
IR service + sealed agents
 │
 ▼
Orchestrator
 │
 ├── enumerate
 ├── extract
 ├── grounding gate
 ├── convert
 ├── validate
 └── commit
 │
 ▼
Results store
 │
 ▼
Streamlit review UI
 │
 ▼
Corrections log
```

The extractor and converter are deliberately separated: the model that reads the paper does not see the schema, while the model that fills the schema does not see the paper. Evidence must therefore be grounded in cited source blocks before it can become a record.

## 2. Core implementation

### Document processing — `src/docproc/`

Marker JSON is converted into `content.md` with stable `⟦b:NNNN⟧` block anchors and a `provenance.json` mapping each anchor to page, section and polygon. A QC gate detects conversions that lose text before extraction begins.

### Extraction pipeline — `src/pipeline/orchestrator.py`

The orchestrator processes entities in dependency order. Candidates are enumerated from prose and, for dense tables, reconstructed deterministically from table structure. Candidates then pass through extraction, grounding, conversion, deterministic validation, readiness classification, bounded AI review and commit.

### IR service

The FastAPI service is the single writer for records. It exposes proposal, commit and unresolved-field operations together with read tools for sections, tables, rows and cells. It enforces the IR schema, provenance labels and reference integrity.

### Deterministic validation and grounding

Validation checks the whole record graph rather than trusting an agent's claim that a record is valid. The grounding gate verifies that quoted evidence actually occurs in the cited source blocks. Null source facts are treated as ungrounded rather than allowing one invalid fact to block an otherwise usable record.

### Table reconstruction

Dense tables are handled through a deterministic pipeline:

```text
A — discover tables
B — reconstruct table role, factors, dimensions, variables and row groups
C — generate candidates from the table structure
D — merge with free-form enumeration
```

The design also handles page-split tables, row-encoded variables, column-encoded factors and pooled factors when supported by the paper.

### Provider resilience

Empty, timed-out and malformed model responses are classified separately from content failures. Provider failures receive their own cooldown/retry handling, and repeated exhausted budgets reduce the run to one round per record so a provider outage does not consume hours of extraction attempts.

### Review UI — `streamlit_app/`

The review interface presents each field as:

**result | source | actions**

A reviewer can approve, edit, relocate evidence, add a note, tell the agent, or open the cited PDF location. Link fields are hidden from review progress, and corrections are appended to an immutable log rather than overwriting the original extraction.

## 3. Prompt changes

The orchestrator prompts were updated for the table-classification, enumeration and conversion stages.

| Prompt | Main change |
|---|---|
| **Table classification** | Requires all table variables, including row-label variables; supplies relevant Methods text for grounding method hints; identifies page-split continuations. |
| **Enumeration** | Uses explicit link-field names, grounds event-condition links in cited text, and can request dimension declarations. |
| **Conversion** | Uses orchestrator-authored candidate context for treatment links and prevents missing page metadata from being incorrectly marked unresolved. |
| **Extraction / AI validation** | Unchanged. |

Additional non-prompt fixes include safer Method matching and dropping null facts without source evidence instead of failing an entire Citation.

## 4. Testing

All tests run without a network or live model. Model calls are replaced by scripted responses and regression fixtures use trimmed paper text and recorded model answers.

| Suite | Result |
|---|---:|
| `src/tests` — 42 modules | **1046 passed** |
| `streamlit_app/tests` — 8 modules | **87 passed, 13 skipped** |
| `src/docproc/test_marker_adapter.py` | **5 passed** |

The skipped Streamlit tests require a browser.

## 5. Real-paper validation

A complete end-to-end run was performed on **Barrios-Masias et al. (2010), “Cultivar mixtures of processing tomato in an organic agroecosystem”**, using `gpt-oss-120b` on the Jetstream endpoint.

| Measurement | Result |
|---|---:|
| Runtime | **3 h 24 min** |
| Model calls | **481** |
| Records | **133** |
| Ready | **96** |
| Unresolved | **35** |
| Error | **2** |
| Provider-failed rounds | **47** |
| Affected records recovered | **12 / 13** |
| Ready observations | **13** |

The 13 ready observation values checked against the paper were correct.

The run also exposed concrete remaining issues:

- The Method pool was incomplete, leaving some observations unresolved rather than incorrectly linked.
- Two Method links were wrong.
- Some Management dates were attached to the wrong events.
- Observation timing in days after planting was not consistently carried into `temporal_info`.
- Treatments were not yet linked to the Study.
- Figures were not extracted because they are image content in the source.

These failures were retained as explicit unresolved/known-limit states rather than silently guessing.

## 6. Reference documents

The IR and extraction targets are based on the **Calibration and Validation Data Collection Protocol** and its `datapackage.json`.

A separate feature request proposes obtaining Citation metadata from DOI/Crossref rather than relying on DOI information being present in the PDF.

## 7. Code

The implementation covers:

- document processing and provenance generation
- IR schema and storage
- deterministic validation and grounding
- FastAPI IR service and sealed agents
- extraction orchestration
- deterministic table enumeration
- provider resilience and run isolation
- Streamlit review and correction workflows

**Code, branches and pull requests:**  
https://github.com/PecanProject/sage/pulls

The original research-paper PDFs, Marker JSON, generated results and logs are not part of the delivered code.

## 8. Known limits and next steps

1. Stabilize Method enumeration and improve Method matching for short names.
2. Fix Management date parsing and consistently carry DAP into observation timing.
3. Link Treatments to Studies and represent factorial designs more completely.
4. Add DOI/Crossref Citation metadata lookup.
5. Improve provider-timeout handling and reduce runtime spent waiting for provider cooldowns.
6. Expand evaluation to a scored gold dataset and additional validation papers.
