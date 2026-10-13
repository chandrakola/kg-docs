# REAL-KG Technical Architecture, BioNLP & Graph Integration Briefing

**Document Identifier:** `REAL-KG-BRIEFING-2026-10-13`  
**Organization:** National Institutes of Health (NIH) / National Library of Medicine (NLM) / LHNCBC  
**Branch:** Applied Clinical Informatics Branch  
**Prepared By:** Chandra Kola  
**Target Audience:** Dr. Mehmet Kayaalp, Kathy, NLM Core Team, K3 Vendor Engineering Team  
**Standard Adherence:** LinkML 0.8.0, Biolink Model 4.3.6, W3C OWL 2, SPARQL 1.1, Cypher  

---

## 1. Executive Summary

**REAL-KG** (Real-World Evidence & Adverse Event Knowledge Graph) is NLM's pharmaceutical knowledge network designed to bridge regulatory label assertions, peer-reviewed medical publications, post-marketing safety signals, and longitudinal real-world health data into an explainable, clinically auditable knowledge graph.

### Architectural Core Principles
1. **Strict Separation of Audit from Serving**:
   - All extracted sentences, entity identifications, stage events, and transformation lifecycles are immutably persisted in **PostgreSQL** (`lab_db`) and **MinIO**.
   - Graph databases (**Neo4j** and **Apache Jena Fuseki**) serve as high-performance query projections, not primary storage.
2. **Deterministic BioNLP Grounding (Zero Hallucination)**:
   - LLMs are never trusted as black boxes.
   - Biomedical sentence boundary detection (`pySBD`), entity recognition (`scispaCy`), clinical negation detection (`negspacy`), fuzzy evidence verification (`rapidfuzz` score $\ge 80\%$), and multi-tier concept normalization ensure that every assertion in the graph is anchored to an exact verbatim source text span.
3. **Dual-Graph Serving Parity**:
   - High-performance Property Graph (**Neo4j LPG**) for multi-hop graph algorithms and Natural Language Querying (NLQ via Bedrock Claude / Gemini).
   - Standard Semantic Web projection (**W3C RDF/SPARQL** via Apache Jena Fuseki) using **W3C OWL 2 Punning** for K3 and external federation.

---

## 2. Supported Clinical Use Cases & Competency Questions

| CQ ID | Clinical Use Case & Question | Harmonized Sources | Graph Capabilities Required |
| :--- | :--- | :--- | :--- |
| **CQ-1** | **Regulatory vs. Literature Discrepancies**<br>*"What adverse reactions or contraindications are documented in peer-reviewed PubMed trials that are not yet incorporated into FDA DailyMed package inserts?"* | DailyMed + PubMed | Predicate alignment (`biolink:has_adverse_event`), source discrepancy diffing, exact verbatim quote citations. |
| **CQ-2** | **Pharmacologic Class Adverse Effect Profiling**<br>*"What class-wide adverse events are shared across all GLP-1 receptor agonists (e.g., pancreatitis, thyroid C-cell tumors) versus compound-specific events?"* | DailyMed + RxClass / MED-RT | OWL 2 Punning class hierarchy (`rdfs:subClassOf`), multi-hop Cypher traversals across EPC/MoA/PE nodes. |
| **CQ-3** | **Mechanism-of-Action (MoA) Hierarchies**<br>*"Which drug products share an identical molecular target or physiologic effect axis despite different approved therapeutic indications?"* | DailyMed + RxNorm + UMLS | Cross-vocabulary normalization (RxCUI to MED-RT / ATC), semantic typing via `TUIClassifier`. |
| **CQ-4** | **Spontaneous Safety Signals vs. Real-World Incidence**<br>*"How does the spontaneous reporting ratio (FAERS PRR/ROR) for acute pancreatitis compare against longitudinal incidence rates in EHR cohorts (CPRD Aurum)?"* | FAERS + CPRD + DailyMed | Statistical association modeling (`biolink:associated_with`) with report count and cohort-history qualifiers. |
| **CQ-5** | **Discrete Conditionality & Adjunct Indications**<br>*"Which drugs are indicated strictly as an adjunct to diet and exercise for glycemic control in Type 2 Diabetes vs. approved for monotherapy or pediatric populations?"* | DailyMed (SPL Text) | First-class edge qualifiers (`condition_qualifier: adjunct_to_diet_and_exercise`, `population_qualifier: adults_and_pediatrics_ge_10`). |

---

## 3. The Four Primary Knowledge Sources

1. **DailyMed (FDA Structured Product Labels - SPL)**:
   - Official FDA regulatory labels in XML format.
   - Filtered to innovator/reference products (NDAs, BLAs).
   - Ingested by LOINC section codes: Boxed Warning (`34066-1`), Indications & Usage (`34067-9`), Adverse Reactions (`34084-4`), Contraindications, Warnings, and Drug Interactions.
2. **PubMed Literature**:
   - Peer-reviewed MEDLINE abstracts and PMC full-text XML.
   - Ingested via curated cohort PMIDs or targeted MeSH queries.
   - Focuses on empirical *Results* and *Conclusions* sections with quotation-grounded evidence.
3. **FAERS (FDA Adverse Event Reporting System)**:
   - Quarterly FDA spontaneous safety report data.
   - Modeled strictly as non-causal statistical associations (`biolink:associated_with`) carrying case counts and reporting ratios, keeping them distinct from FDA label warnings.
4. **CPRD Aurum (UK Primary Care EHR Data)**:
   - Longitudinal electronic health records containing Protected Health Information (PHI).
   - **Governed strictly offline/ad hoc by Dr. Mehmet Kayaalp**. No patient-level records are exposed to graph databases or external LLM APIs; only approved aggregate statistical profiles enter the pipeline.

---

## 4. Dual-Graph Topology & Serving Architecture

```
[Raw Sources: DailyMed, PubMed, FAERS, CPRD]
                     │
                     ▼
       ┌───────────────────────────┐
       │     PIPELINE CORE (uv)    │
       │  1. source-nexus          │
       │  2. assertion-engine      │
       │  3. concept-align         │
       │  4. biolink-bridge        │
       └─────────────┬─────────────┘
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
┌──────────────────┐    ┌───────────────────────────────────┐
│ POSTGRESQL AUDIT │    │ DUAL SERVING PROJECTIONS          │
│ • triples (18.4k)│    │                                   │
│ • provenance     │    │ • Neo4j LPG (7,094 N, 19,430 E)   │
│   (16.3k)        │    │   (Cypher, NLQ, Insight Studio)   │
│ • stage_events   │    │                                   │
│   (96.8k)        │    │ • Apache Jena Fuseki RDF          │
│ • UMLS (105.8M)  │    │   (46.8k Triples, OWL 2 Punning,  │
│                  │    │    SPARQL Endpoint for K3)        │
└──────────────────┘    └───────────────────────────────────┘
```

---

## 5. End-to-End Pipeline Stage Workflow

* **Stage 1 (`subprojects/source-nexus`)**: SPL XML discovery, parsing, and JSON transformation by LOINC sections. Produces `structured_base/*.json`.
* **Stage 2 (`subprojects/assertion-engine`)**: `pySBD` sentence segmentation + LLM SPO extraction with strict JSON schemas. Emits triples with sentence offsets into PostgreSQL.
* **Stage 3 (`subprojects/concept-align`)**: 4-Tier normalization mapping entities to RxNorm, UMLS, MeSH, SNOMED-CT, MedDRA, and MONDO. Resolves pharmacologic classes (EPC, MoA, PE, CS via MED-RT) and ATC.
* **Stage 4 (`subprojects/biolink-bridge`)**: Emits the 23-column KGX wire format (`nodes.tsv`, `edges.tsv`) and reified W3C OWL 2 RDF (`semantic_graph.nt`).
* **Stage 5 (`subprojects/quality-guardian`)**: Enforces fail-closed contract validations (`validate_contracts.py`), quarantines discrepancies, and triggers atomic loads into Neo4j and Fuseki.

---

## 6. Hybrid BioNLP & Concept Normalization Engine

Instead of relying on LLMs as unstructured text generators, REAL-KG couples classical BioNLP tools with generative models:

1. **Biomedical Sentence Boundary Disambiguation (`pySBD`)**:
   - `pysbd.Segmenter(language="en", clean=True)` protects decimal numbers (`0.5 mg`), percentages (`1.5%`), scientific abbreviations (`Dr.`, `et al.`), and p-values (`p<0.001`), producing clean numbered sentences (`S1.`, `S2.`).
2. **Biomedical NER & Negation (`scispaCy` & `negspacy`)**:
   - `en_ner_bc5cdr_md` extracts specific chemical and disease mentions.
   - `negspacy` (`en_clinical` termset) identifies clinical negation to ensure negative findings (e.g., *"no pancreatitis observed"*) are classified as contraindications or omitted rather than parsed as positive indications.
3. **Fuzzy Evidence Grounding (`rapidfuzz`)**:
   - Every candidate evidence quote is verified against source sentences using `fuzz.partial_ratio`.
   - Threshold $\ge 80\%$ required; ungrounded assertions are automatically quarantined.
4. **4-Tier Normalization Hierarchy**:
   - **Tier 0:** Local PostgreSQL UMLS Metathesaurus (`mrconso`, `mrsat` with 105.8M records). Instantaneous, offline.
   - **Tier 1:** Local RxNav (`localhost:4000/REST`) for active ingredients, RxCUIs, MED-RT classes, and ATC codes.
   - **Tier 2:** Cambridge **SapBERT** (`cambridgeltl/SapBERT-from-PubMedBERT-fulltext`) for transformer embedding cosine similarity ($\ge 0.70$).
   - **Tier 3/4:** scispaCy linkers and remote NLM UTS REST API fallback.

---

## 7. LLM Orchestration, Caching & Safety Controls

1. **Deterministic Structured JSON Schemas**:
   - Extractions enforce `response_mime_type: "application/json"` with `responseJsonSchema` or Pydantic models.
   - `temperature = 0` and `GEMINI_THINKING_BUDGET = 0` for reproducible, low-latency extractions.
2. **Persisted Disk Caching (`diskcache`)**:
   - Prompt/response pairs are stored on disk under `application/cache` (`ASSERTION_LLM_CACHE=on`).
   - Keyed on `sha256(prompt + provider + model + temperature)`. Identical rerun executions incur zero API costs.
3. **Batch API & Quota Coordination**:
   - Google Gemini Batch API (`GOOGLE_API_BATCH=files:on`) with MongoDB-backed atomic locks (`google_batch_limiter.py`) preventing 429 quota exhaustion.
4. **Read-Only NLQ Guardrails**:
   - Natural Language Query (NLQ via Bedrock Claude Opus / Gemini) translates user questions to Cypher.
   - Read-only driver enforcement (`READ_ACCESS`) and AST validation reject mutating operations (`CREATE`, `DELETE`, `SET`).

---

## 8. K3 Vendor Integration & Semantic Web (RDF/OWL) Standards

To ensure complete compatibility with K3's RDF-based knowledge graph implementation at NLM:

1. **W3C OWL 2 Punning**:
   - Substance-grouping classes (EPC, MoA, CS, ATC, Chemical) are declared with dual typing: `owl:Class` and `owl:NamedIndividual`, decorated with `schema:DrugClass`.
   - Enables drug classes to act simultaneously as semantic types in reasoners and as navigable nodes in graph queries.
2. **Direct Subclass Materialization**:
   - Materializes direct `rdfs:subClassOf` triples for all 42 DailyMed taxonomic edges and 7 PubMed relationships.
3. **Reified Statements for Provenance**:
   - Edge qualifiers, clinical evidence text, confidence metrics, and `biolink:primary_knowledge_source` CURIEs are attached via reified statement nodes (`rdf:Statement` / `biolink:Association`).
4. **SPARQL 1.1 Endpoint**:
   - Deployed to Apache Jena Fuseki (`/rwdkn` dataset), fully queryable via standard W3C SPARQL.

---

## 9. Update Cadence & Human-in-the-Loop (HITL) Governance

* **DailyMed**: Automated batch runs executed periodically against updated monthly FDA SPL release archives.
* **PubMed**: Incremental batch runs triggered against newly indexed PMIDs matching monitoring bibliographies.
* **FAERS**: Automated quarterly runs aligned with FDA’s quarterly AERS/FAERS data releases.
* **CPRD Aurum**: Ad hoc / offline governance by Dr. Kayaalp; releases remain in `DRAFT` until explicit SME clearance.
* **Human-in-the-Loop Review Surfaces**:
  - **Triple Audit Explorer (`/demo/triples/`)**: Clinicians review accepted, skipped, and quarantined candidate triples.
  - **Statement Provenance Explorer (`/demo/sentences/`)**: Inspects verbatim clinical quotes and confidence scores before promotion to the live serving stores.

---

## 10. Audit & Evidence Bedrock (PostgreSQL & MinIO)

The authoritative evidence bedrock is maintained in PostgreSQL (`lab_db`):

| Table Name | Verified Records | Primary Key & Key Columns | Functional Purpose |
| :--- | ---: | :--- | :--- |
| `triples` | 18,415 | `triple_id`, `subject`, `predicate`, `object`, `sentence_id` | Canonical registry of extracted SPO assertions with LinkML slots. |
| `triple_provenance` | 16,293 | `sentence_id`, `evidence`, `source_url`, `section_id` | Verbatim clinical text and document offsets for regulatory audit. |
| `sentence_stage_events`| 96,831 | `event_id`, `stage`, `status`, `timestamp` | Chronological lifecycle log tracking extraction, normalization, and export. |
| `mrconso` / `mrsat` | 105.8M | `cui`, `str`, `sab`, `tty`, `code` | UMLS Metathesaurus local lookup database powering Tier-0 normalization. |

### Sample Provenance SQL Query
```sql
SELECT 
    t.subject, t.predicate, t.object, 
    p.evidence AS raw_clinical_sentence, 
    p.source_url, p.section_id
FROM triples t
JOIN triple_provenance p ON t.sentence_id = p.sentence_id
WHERE t.subject = 'RXCUI:1991302' -- Semaglutide
LIMIT 5;
```
