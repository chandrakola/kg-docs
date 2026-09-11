# REAL-KG / RWDKN: Ground Truth Reviewed Data Integration & Utilization Plan

> **Date:** September 11, 2026  
> **Author / Project:** RWDKN Pipeline Engineering Team  
> **Status:** Proposal / Review Document for Architecture Review  
> **Target Systems:** `rwdkn-data-pipeline`, `rwdkn-service`, `mylab` (`kolaci9ubuntu`)

---

## 1. Executive Summary & Context

During the evolution of the Real-World Data Knowledge Network (REAL-KG / RWDKN), an SME (Subject Matter Expert) curation workbench was established to validate relation extraction from clinical and scientific literature (DailyMed structured product labels and PubMed articles). 

Subject matter experts reviewed candidate triples and statements, producing a **curated Gold Set** of approved, rejected, and flagged triples.

This document serves as an architectural assessment and implementation plan to:
1. Document **where and how** this reviewed ground truth data is currently preserved.
2. Identify the **integration gaps** between the curation workbench and the downstream knowledge graph pipeline.
3. Provide a step-by-step **implementation roadmap** to integrate this data into Quality Guardian, Biolink Bridge (KGX export), and LLM prompt calibration.
4. Supply structured context for **peer review by an external AI tool or architect**.

---

## 2. Current Preservation & Storage Audit

### A. Is the reviewed ground truth data safe?
**Yes.** The data is preserved in version control (Git) and on persistent runtime file systems. It is protected against automated pipeline clean-wipes, though it has not yet been loaded into a centralized relational database.

### B. Storage Locations

| Layer | Environment | File Path / Storage Target | Description |
| :--- | :--- | :--- | :--- |
| **Tracked Gold Standard (Git)** | Code Repository | `rwdkn-data-pipeline/subprojects/quality-guardian/fixtures/ground-truth/reviews.jsonl` | **Primary Source of Truth:** 1.33 MB append-only log containing 240 curated SME records. Committed in Git. |
| **Service Bootstrap Seed (Git)** | Code Repository | `rwdkn-service/services/ground-truth/seed/reviews.seed.jsonl` | Committed seed file used to initialize the Ground Truth Review API when spun up on new nodes. |
| **Live Runtime Volume** | Production (`mylab`) | Named Volume: `rwdkn-ground-truth-data`<br>Host: `/var/lib/docker/volumes/rwdkn-ground-truth-data/_data/reviews.jsonl`<br>Container: `/srv/real-kg/services/ground-truth/data/` | Live volume mounted to `real-kg-ground-truth:8091`. All new curator submissions append here in real-time. |
| **Materialized Cache** | Runtime | `state.json` (inside volume / data dir) | Fast in-memory/disk index of the latest decision per `candidate_id`, derived dynamically by replaying `reviews.jsonl`. |
| **Manual Triples Log** | Runtime & Repo | `manual_triples.jsonl` | Append-only log for novel triples authored directly by SMEs during curation. |

### C. Data Invariant & Durability Design
In commit `49c420f` (*"refactor(ground-truth): relocate SME curation to tracked fixtures path"*), the curation files were deliberately relocated out of `application/ground_truth/` into `subprojects/quality-guardian/fixtures/ground-truth/` so that automated pipeline clean runs (`cleanup_runtime.sh` / `rm -rf application/logs/*`) **never wipe curated SME decisions**.

---

## 3. Data Schema: What Does a Reviewed Record Look Like?

Each record in `reviews.jsonl` is a JSON line containing:
1. **The Review Decision**: `decision` (`approved`, `rejected`, `needs_discussion`), `reviewer`, `timestamp`, `note`, and any manual field corrections (`corrected_subject`, `corrected_predicate`, `corrected_object`).
2. **The Candidate Statement / Triple**: Source document metadata (PubMed `pmid`, DailyMed `setid`), exact text fragment (`statement_url`), confidence score, extracted triple (`subject`, `predicate`, `object`), and evidence grounding.

### Example Record from `reviews.jsonl`:
```json
{
  "type": "sme_review",
  "candidate_id": "gt:beb13766ffcaf42e1ba4e645",
  "decision": "approved",
  "reviewer": "SME",
  "note": "Validated clinical trial weight reduction endpoint.",
  "timestamp": "2026-06-08T22:48:21.715345+00:00",
  "candidate": {
    "document_type": "pubmed",
    "document_id": "39761578",
    "candidate_kind": "statement",
    "subject": "tirzepatide",
    "predicate": "decreases",
    "object": "body weight",
    "confidence": 0.9,
    "evidence": "Compared with placebo, tirzepatide (15 mg once weekly) resulted in weight loss of up to 17.8% after 72 weeks...",
    "source_url": "https://pubmed.ncbi.nlm.nih.gov/39761578/",
    "statement_url": "https://pubmed.ncbi.nlm.nih.gov/39761578/#:~:text=Compared%20with%20placebo...",
    "triple_id": "PUBMED:39761578:abstract:6016637b0a0f563e"
  }
}
```

---

## 4. Current Integration Gap Analysis

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                               CURRENT ARCHITECTURE                                │
└───────────────────────────────────────────────────────────────────────────────────┘

 [DailyMed / PubMed]
         │
         ▼
 [assertion-engine] ────► [triple_audit.jsonl] ────► [Ground Truth API :8091]
                                                             │
                                                             ▼
                                                    [reviews.jsonl] (1.33 MB)
                                                    [SME Gold Set Fixture]
                                                             │
                                                             ▼
                                                    ┌─────────────────┐
                                                    │  GAP DETECTED   │
                                                    │  (Isolated Set) │
                                                    └────────┬────────┘
                                                             │ ❌ NOT CONNECTED
                                                             ▼
 ┌─────────────────────────────────────────────────────────────────────────────────┐
 │ 1. quality-guardian   : No automated precision/recall score against Gold Set    │
 │ 2. biolink-bridge     : Approved triples NOT injected into final KGX TSVs       │
 │ 3. assertion-engine   : Rejected triples NOT suppressed; no few-shot feedback   │
 │ 4. Neo4j / Postgres   : Not loaded into graph or relational tables              │
 └─────────────────────────────────────────────────────────────────────────────────┘
```

* **What IS integrated today:**
  * Candidates stream into the review workbench.
  * SMEs can view evidence, approve/reject statements, author manual triples, and export JSON via `GET /api/ground-truth/export`.
* **What is MISSING:**
  * **No Pipeline Feedback Loop:** Curated decisions do not influence subsequent ETL runs.
  * **No Knowledge Graph Injection:** Approved triples exist only in the review JSONL, not in the production Neo4j graph.
  * **No CI Regression Gate:** PRs and pipeline runs do not measure extraction accuracy against the gold set.

---

## 5. Five-Phase Implementation Plan

### Phase 1: Automated Quality & Regression Gate (`quality-guardian`)
**Objective:** Calculate automated Precision, Recall, and F1 metrics for LLM extraction on every pipeline release.

1. **Create Evaluation Script:** `subprojects/quality-guardian/scripts/evaluate_extraction_accuracy.py`.
2. **Logic:**
   * Load `subprojects/quality-guardian/fixtures/ground-truth/reviews.jsonl`.
   * Partition records into `GOLD_APPROVED` and `GOLD_REJECTED`.
   * Compare against current run's `application/logs/triples/triple_audit.jsonl` (or Stage 4 KGX edges).
   * Calculate:
     $$\text{Precision} = \frac{\text{Extracted} \cap \text{GOLD\_APPROVED}}{\text{Extracted}}$$
     $$\text{Hallucination Rate} = \frac{\text{Extracted} \cap \text{GOLD\_REJECTED}}{\text{Extracted}}$$
3. **CI Integration:** Add a gate in `Jenkinsfile` / GitHub Actions:
   ```bash
   uv run pytest subprojects/quality-guardian/tests/test_ground_truth_benchmark.py
   ```
   *Fail build if Hallucination Rate > 5% or Precision < 90%.*

---

### Phase 2: Graph Enrichment & Injection (`biolink-bridge`)
**Objective:** Guarantee that all SME-approved and manually authored triples are present in the final Neo4j Knowledge Graph.

1. **Loader Step in Stage 4 (`biolink-bridge`):**
   * Before exporting `kgx_edges.tsv`, read:
     * `fixtures/ground-truth/reviews.jsonl` (filter: `decision == "approved"`)
     * `manual_triples.jsonl`
2. **Provenance & Evidence Tagging:**
   * Assign edge properties:
     * `knowledge_level = "curated"`
     * `agent_type = "manual_validation"`
     * `curated_by = review.reviewer`
     * `sme_decision_timestamp = review.timestamp`
3. **Deduplication:** Merge with automated extraction edges, overriding automated confidence with `1.0` (SME verified).

---

### Phase 3: Negative Filtering & Suppression Gate
**Objective:** Prevent known, previously rejected LLM hallucinations from re-entering the graph.

1. **Build Suppression Index:** Compile a bloom filter or hash set of all `GOLD_REJECTED` `(subject, predicate, object)` triples and `(pmid, sentence_id)` pairs.
2. **Filter at Stage 2 / Stage 3:**
   * In `subprojects/concept-align` or `biolink-bridge`, if an incoming candidate matches a rejected gold triple:
     * Drop the candidate.
     * Route to `application/logs/triples/triple_suppressed.jsonl` with reason: `SME_REJECTED_GOLD_SET`.

---

### Phase 4: Prompt Engineering & In-Context Few-Shot Calibration
**Objective:** Improve raw LLM assertion extraction accuracy by injecting SME reviews directly into model prompts.

1. **Exemplar Extraction:** Extract top 10 subtle approved triples and top 10 common rejected errors from `reviews.jsonl`.
2. **Prompt Injection:**
   * Update `CYPHER_TRANSLATOR_PROMPT` in `query-app` and Gemini/Claude extraction prompts in `assertion-engine`.
   * Add dedicated Few-Shot sections:
     ```markdown
     ### Positive Exemplars (SME Approved):
     - Statement: "Tirzepatide resulted in weight loss of up to 17.8%"
       Triple: (tirzepatide, decreases, body weight) [Approved]

     ### Negative Exemplars (Common Hallucinations - DO NOT EXTRACT):
     - Statement: "Patients with prior GLP-1 exposure were excluded"
       Triple: (GLP-1, treats, exclusion criteria) [REJECTED by SME: Protocol eligibility, not medical relation]
     ```

---

### Phase 5: Database Persistence & Versioning (Long-Term)
**Objective:** Upgrade from flat JSONL files to a relational schema in PostgreSQL for multi-curator scaling.

1. **Create PostgreSQL Schema in `rwdkn-service`:**
   ```sql
   CREATE TABLE curation.sme_reviews (
       candidate_id VARCHAR(64) PRIMARY KEY,
       document_type VARCHAR(32) NOT NULL,
       document_id VARCHAR(64) NOT NULL,
       subject VARCHAR(255) NOT NULL,
       predicate VARCHAR(128) NOT NULL,
       object VARCHAR(255) NOT NULL,
       decision VARCHAR(32) NOT NULL CHECK (decision IN ('approved', 'rejected', 'needs_discussion')),
       reviewer VARCHAR(128) NOT NULL,
       note TEXT,
       evidence TEXT,
       created_at TIMESTAMPTZ NOT NULL,
       updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
   );
   CREATE INDEX idx_curation_doc ON curation.sme_reviews (document_type, document_id);
   CREATE INDEX idx_curation_decision ON curation.sme_reviews (decision);
   ```
2. **Migration Script:** Add `tools/curation/migrate_jsonl_to_postgres.py` to ingest `reviews.jsonl` and `manual_triples.jsonl` into the database.

---

## 6. Review Questions for External AI / Peer Reviewer

When reviewing this plan with another AI tool or software architect, consider evaluating the following questions:

1. **Granularity & Match Keys:** Is matching on `(subject, predicate, object)` sufficient for suppression, or should suppression be scoped to `(document_id, subject, predicate, object)` to avoid suppressing valid relations in different contexts?
2. **Biolink Compliance:** When injecting manual or approved triples into `kgx_edges.tsv`, how should Biolink Model 3.x+ handle the provenance fields (`knowledge_level`, `agent_type`, `has_evidence`) to ensure LinkML validation passes?
3. **Conflict Resolution:** If an automated extraction run changes its confidence score or sentence grounding for an existing `candidate_id`, how should the materialized `state.json` handle the merge?
4. **Data Sync Architecture:** Since `reviews.jsonl` lives in `rwdkn-data-pipeline` and the running service lives in `rwdkn-service` (and on `mylab`), what is the cleanest automated CI/CD mechanism to sync new reviews from the live Docker volume back into the Git repository?

---

## 7. Reference File Index

* **Curated Gold Set (Git):** [`rwdkn-data-pipeline/subprojects/quality-guardian/fixtures/ground-truth/reviews.jsonl`](file:///Users/kolac/Developer/NLM/kg/rwdkn-data-pipeline/subprojects/quality-guardian/fixtures/ground-truth/reviews.jsonl)
* **Service Seed File (Git):** [`rwdkn-service/services/ground-truth/seed/reviews.seed.jsonl`](file:///Users/kolac/Developer/NLM/kg/rwdkn-service/services/ground-truth/seed/reviews.seed.jsonl)
* **Ground Truth API Service:** [`rwdkn-service/shared/ground_truth_api.py`](file:///Users/kolac/Developer/NLM/kg/rwdkn-service/shared/ground_truth_api.py)
* **Ground Truth Workbench UI:** [`rwdkn-service/docker/ground-truth/index.html`](file:///Users/kolac/Developer/NLM/kg/rwdkn-service/docker/ground-truth/index.html)
* **Workbench Specification:** [`rwdkn-data-pipeline/docs/specs/ground-truth-review-workbench.md`](file:///Users/kolac/Developer/NLM/kg/rwdkn-data-pipeline/docs/specs/ground-truth-review-workbench.md)
* **Repository Split Architecture:** [`kg-docs/architecture/repository-split.md`](file:///Users/kolac/Developer/NLM/kg/kg-docs/architecture/repository-split.md)
