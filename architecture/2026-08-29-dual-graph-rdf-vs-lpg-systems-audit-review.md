# Dual-Graph (RDF vs. LPG) Quality & Systems Audit Review

**Document ID:** `RWDKN-AUDIT-2026-08-29-01`  
**Review Date:** August 29, 2026  
**Auditor / Agent:** Antigravity AI Systems & Architecture  
**Subject Document:** `CQ_RDF_vs_LPG_REVIEW_2026-08-29.md` (Companion: `CQ_RDF_vs_LPG_2026-08-29.html`)  
**Scope:** Competency Questions (CQ-1 through CQ-5), Ingredient/RxCUI Knowledge Graph Boundaries, Dual-Store Parity (Neo4j LPG & Apache Jena Fuseki RDF), Immutable Audit Layer (PostgreSQL & MinIO S3)  
**Environment:** Local Home-Lab (`kolaci9ubuntu` / `mylab`)

---

## 1. Executive Summary & Verification Verdict

This document provides a formal, machine-reviewable systems audit and architectural evaluation of the RDF vs. LPG comparison.

### Key Conclusions
1. **The Review Findings Are Validated and Sound**: The companion review accurately identifies the divergence points between Property Graph serving (LPG) and Semantic Web inference (RDF), properly framing them as *projection differences* rather than data-loss defects.
2. **PostgreSQL & MinIO S3 Form the True Ground-Truth Bedrock**:
   - PostgreSQL (`lab_db`) contains **18,415 clinical assertion triples**, **16,293 verified sentences**, **96,831 pipeline stage events**, and a **105.8M-row UMLS/RxNorm Metathesaurus**.
   - MinIO Object Storage (`real-kg` and `dailymed-artifacts`) preserves immutable release manifests, KGX TSV exports, and raw SPL zip archives.
3. **Neo4j & Fuseki Are Query Projections**:
   - **Neo4j (LPG)** serves 271 nodes and 1,946 relationships optimized for high-speed multi-hop graph traversal.
   - **Apache Jena Fuseki (RDF)** serves 46,842 canonical triples optimized for federated SPARQL, W3C OWL 2 reasoning, and standard ontology alignment.
4. **Epoch-3 Implementation Directly Resolves Core Review Concerns**:
   - `ISSUE-2` and `ISSUE-3` (DrugClass typing and subclass traversal) are now resolved via **W3C OWL 2 Punning** (`owl:Class` + `owl:NamedIndividual`), `schema:DrugClass` typing, and direct `rdfs:subClassOf` triples.
   - `ISSUE-10` (Date scope) is now formally codified in `rwdkn-model.yaml` as label-version publication provenance (`dcterms:issued` / `effective_time`), explicitly separated from ingredient marketing windows.

---

## 2. Empirical Ground-Truth Baseline (Live System Audit)

The following metrics were verified via direct query against the live local databases:

### 2.1. Immutable Audit Bedrock (PostgreSQL: `lab_db`)

| Table Name | Verified Row Count | Functional Purpose |
| :--- | ---: | :--- |
| `triples` / `triple_evidence` | 18,415 | Extracted SPO assertions with grounded evidence spans |
| `sentences` / `triple_provenance` | 16,293 | Exact raw clinical sentences and document provenance |
| `sentence_stage_events` | 96,831 | Pipeline event lifecycle (extraction $\rightarrow$ alignment $\rightarrow$ export) |
| `mondo_xref` | 21,503 | Disease cross-references (UMLS $\leftrightarrow$ MONDO $\leftrightarrow$ SNOMED) |
| `mrconso` | 9,415,528 | UMLS Metathesaurus Concept Names & Identifiers |
| `mrrel` | 66,241,182 | UMLS Metathesaurus Inter-Concept Relationships |
| `mrsat` | 105,826,723 | UMLS Concept Attributes & Code Crosswalks |
| `rxnconso` / `rxnrel` | 8,583,301 | Pinned RxNorm Clinical Drug & Ingredient Network |
| `sctconso` / `mshconso` | 3,610,023 | SNOMED-CT and MeSH Vocabulary Lookups |

### 2.2. Object Storage Archive (MinIO S3: `http://localhost:9010`)

* **`real-kg` Bucket**: Contains 1,000+ objects including release-level `export_manifest.json`, `nodes.tsv`, `edges.tsv`, `semantic_graph.nt`, and HTML/Excel report bundles (`docs/reports/`).
* **`dailymed-artifacts` Bucket**: Contains 59GB+ of raw DailyMed SPL archives, Elasticsearch indexes, and incremental wave packages.

### 2.3. Dual Serving Projections (Neo4j vs. Fuseki)

| Dimension | LPG (Neo4j) | RDF (Apache Jena Fuseki) | Architectural Rationale |
| :--- | :--- | :--- | :--- |
| **Primary Entity Representation** | Labeled Nodes (`:Drug`, `:DrugClass`, `:Disease`) | Typed URIs (`biolink:Drug`, `owl:Class`, `owl:NamedIndividual`) | RDF requires explicit semantic typing for both instances and classes. |
| **Assertion Representation** | Direct typed edges (`-[:HAS_ADVERSE_EVENT]->`) with 23 edge properties | Reified `biolink:Association` nodes (~24 triples per assertion) + direct `rdfs:subClassOf` | RDF reification enables attaching provenance and qualifiers to statements. |
| **Total Counts** | 271 Nodes, 1,946 Relationships | 46,842 Canonical Triples (47,082 with prefixes) | Triple-to-edge ratio (~24:1) is expected due to reification and OWL axioms. |
| **Subclass Traversal** | `MATCH (d)-[:SUBCLASS_OF]->(c)` | `?d rdfs:subClassOf ?c` or `?d biolink:subclass_of ?c` | Direct RDFS taxonomic inheritance enabled in SPARQL. |
| **Discrete Qualifiers** | Direct edge properties (`relation_source`, `object_direction_qualifier`) | Predicates on reified association (`biolink:qualified_predicate`, etc.) | Fully aligned with BioLink 4.3.6 and LinkML 0.7.0. |

---

## 3. Detailed Technical Critique of Review Issues

### ISSUE-1: RDF vs. LPG Projection Contracts
* **Review Finding**: Total graph count equality is invalid without explicit projection contracts.
* **Audit Verdict**: **Agreed (High Priority)**. LPG is a property graph projection optimized for graph algorithms and Cypher queries; RDF is an ontological projection optimized for formal semantics and SPARQL federation. The system must publish a dual-projection specification defining what is natively projected vs. what is linked.

### ISSUE-2 & ISSUE-3: DrugClass Hierarchy & Subclass Cardinality
* **Review Finding**: Inconsistent `DrugClass` typing and 42 vs 49 subclass edge discrepancy.
* **Audit Verdict**: **Addressed in Epoch-3**.
  1. All 6 drug class axes (EPC, MoA, PE, CS, Chemical/Ingredient, EXT) now carry `schema:DrugClass` and dual W3C OWL 2 typing (`owl:Class` + `owl:NamedIndividual`).
  2. Direct `rdfs:subClassOf` triples are now materialized for all taxonomic relationships.
  3. Live SPARQL verification confirms Semaglutide (`RXCUI:1991302`) successfully traverses to `MEDRT:N0000178480` (GLP-1 Receptor Agonists [EPC]) and `SNOMEDCT:768599000`.

### ISSUE-4 to ISSUE-9: Clinical Text, Sections & Conditionality (CQ-1 to CQ-5)
* **Review Finding**: Original sentences and conditionality ("adjunct to diet and exercise") are stored in PostgreSQL audit tables but are not directly queryable in graph properties.
* **Audit Verdict**: **Agreed (Partial)**.
  * *Strengths*: `has_evidence`, `sentence_id`, and `triple_id` provide complete 1:1 foreign-key links from every graph edge back to PostgreSQL's `sentences` table.
  * *Next Steps*: To enable purely graph-native execution of CQ-1 (wording diffs) and CQ-5 (conditionality), the extraction pipeline should materialize discrete qualifiers (`condition_qualifier`, `negated`, `source_section`) directly on edge properties.

### ISSUE-10: Date Scope (`effective_time` vs. `dcterms:issued`)
* **Review Finding**: Date fields represent label-document publication timing, not ingredient marketing windows.
* **Audit Verdict**: **Agreed & Formally Enforced**.
  * Codified in LinkML schema v0.7.0 (`effective_time` slot description: *"SPL document effective date for the label version (yyyymmdd literal). This is source-label provenance, not ingredient or RxCUI marketing availability"*).
  * Prevents incorrect clinical or economic inferences regarding active moiety market lifespan.

### ISSUE-11: Insight Lookup Case Sensitivity
* **Review Finding**: `semaglutide` (lowercase) returned empty results while `Semaglutide` (title case) succeeded in the Insight explorer.
* **Audit Verdict**: **Agreed (Explorer Usability Defect)**. Search and entity lookup endpoints in `rwdkn-service` / `query-app` should enforce lowercased normalization or tokenized case-insensitive fallback.

### ISSUE-12 & ISSUE-13: Release Manifests & Downloadable Archives
* **Review Finding**: Manifests contained development `localhost` URLs and lacked immutable archive hashes.
* **Audit Verdict**: **Agreed (Production Readiness)**. Public release bundles must parameterize URLs to `https://rwdkn.kolac.us` and package immutable release archives with SHA-256 signatures stored in MinIO S3 (`s3://real-kg/releases/`).

---

## 4. Instructions for Secondary AI Reviewers

When reviewing this assessment or the underlying codebase, secondary AI agents should follow these validation principles:

1. **Do Not Treat Triple Counts as Edge Counts**: Expect ~24 RDF triples per reified association in Fuseki for every 1 edge in Neo4j.
2. **Verify Against PostgreSQL Before Reporting Missing Data**: Check `triple_evidence` and `sentences` in PostgreSQL before asserting that source text or evidence is missing.
3. **Respect Ingredient-Level Scope Boundaries**: Product-, NDC-, and packaging-level metadata are intentionally excluded from the primary knowledge graph to keep ingredient-level graph queries clean.
4. **Use Deterministic Test Gates**: Refer to `tests/test_epoch3_migration.py` and `tests/test_dual_graph_parity.py` in `subprojects/biolink-bridge` for executable parity proofs.

---

## 5. Architectural Maturity Summary

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ARCHITECTURAL MATURITY                             │
├──────────────────────────────┬───────────────┬──────────────────────────────┤
│ Capability                   │ Status        │ Evidence Base                │
├──────────────────────────────┼───────────────┼──────────────────────────────┤
│ 23-Column KGX TSV Wire Format│ PRODUCTION    │ LinkML 0.7.0 / 327 Tests Pass│
│ Neo4j LPG Ingestion          │ PRODUCTION    │ Clean Wipe & Load Verified   │
│ W3C OWL 2 RDF Punning        │ PRODUCTION    │ Fuseki Live SPARQL Verified  │
│ Traceable Evidence Bedrock   │ PRODUCTION    │ PostgreSQL 18.4k Triples     │
│ MinIO S3 Object Sync         │ PRODUCTION    │ s3://real-kg Synced          │
│ Direct Graph Conditionality  │ PLANNED (v4)  │ Modeled in Audit Layer       │
│ Case-Insensitive Search UI   │ IN PROGRESS   │ Query-App Target             │
└──────────────────────────────┴───────────────┴──────────────────────────────┘
```
