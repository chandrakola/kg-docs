# Dual-Graph (RDF vs. LPG) Quality & Systems Audit Review

**Document ID:** `RWDKN-AUDIT-2026-08-29-01-REV2`  
**Revision Date:** August 29, 2026  
**Document Role:** Architectural Assessment & Systems Review Response (Feedback on `/home/chandrakola/CQ_RDF_vs_LPG_REVIEW_2026-08-29.md`)  
**Scope:** Competency Questions (CQ-1 through CQ-5), Ingredient/RxCUI Knowledge Graph Boundaries, Dual-Store Parity (Neo4j LPG & Apache Jena Fuseki RDF), Audit Layer (PostgreSQL & MinIO S3)  
**Environment:** Local Home-Lab (`kolaci9ubuntu` / `mylab`)

---

## 1. Executive Summary & Verification Verdict

This document provides a refined, calibrated systems assessment and architectural critique of the RDF vs. LPG comparison.

### Key Conclusions
1. **The Review Findings (Review A) Are Substantially Sound**: Review A accurately identifies divergence points between Property Graph serving (LPG) and Semantic Web inference (RDF), properly framing them as *projection differences* rather than data-loss defects.
2. **Audit Bedrock vs. Serving Projections**:
   - **PostgreSQL (`lab_db`) & MinIO**: Serve as the authoritative audit store containing **18,415 clinical assertion triples**, **16,293 provenance records** (`triple_provenance`), **96,831 pipeline stage events**, and a **105.8M-row UMLS/RxNorm Metathesaurus**. MinIO holds raw SPL tarballs and generated release bundles.
   - **Neo4j (LPG)**: Houses the multi-source serving graph (7,094 nodes, 19,430 relationships globally; 271 nodes, 1,946 edges for DailyMed single-source load) optimized for multi-hop traversals and Cypher queries.
   - **Apache Jena Fuseki (RDF)**: Houses the canonical Semantic Web projection (`/rwdkn` dataset) generated from `semantic_graph.nt` (46,842 N-Triples for DailyMed), supporting SPARQL queries and W3C OWL 2 punning.
3. **Epoch-3 LinkML 0.7.0 Implementation Progress**:
   - `ISSUE-2` and `ISSUE-3` (DrugClass typing and subclass traversal) are implemented via **W3C OWL 2 Punning** (`owl:Class` + `owl:NamedIndividual`), `schema:DrugClass` typing for substance-grouping axes, and direct `rdfs:subClassOf` triples.
   - `ISSUE-10` (Date scope) is formally codified in `rwdkn-model.yaml` as label-version publication provenance (`dcterms:issued` / `effective_time`), explicitly separated from ingredient marketing windows.
   - `ISSUE-11` (Insight Case Sensitivity) is resolved in `subprojects/insight-studio/server/app.js` using case-insensitive name matching (`toLower(s.name) = toLower($name)`) and synonym aliases.

---

## 2. Empirical Ground-Truth Baseline (Live System Audit)

### 2.1. Audit & Provenance Bedrock (PostgreSQL: `lab_db`)

| Table Name | Verified Row Count | Functional Purpose |
| :--- | ---: | :--- |
| `triples` | 18,415 | Extracted SPO assertions with grounded evidence spans |
| `triple_provenance` | 16,293 | Exact raw clinical sentences and document provenance |
| `sentence_stage_events` | 96,831 | Pipeline event lifecycle (extraction $\rightarrow$ alignment $\rightarrow$ export) |
| `mondo_xref` | 21,503 | Disease cross-references (UMLS $\leftrightarrow$ MONDO $\leftrightarrow$ SNOMED) |
| `mrconso` | 9,415,528 | UMLS Metathesaurus Concept Names & Identifiers |
| `mrrel` | 66,241,182 | UMLS Metathesaurus Inter-Concept Relationships |
| `mrsat` | 105,826,723 | UMLS Concept Attributes & Code Crosswalks |
| `rxnconso` / `rxnrel` | 8,583,301 | Pinned RxNorm Clinical Drug & Ingredient Network |
| `sctconso` / `mshconso` | 3,610,023 | SNOMED-CT and MeSH Vocabulary Lookups |

### 2.2. Dual Serving Projections: Source-Scoped vs. Global Multi-Source Counts

To avoid conflating single-source export artifacts with the multi-source serving graph, counts are categorized explicitly:

| Metric / Dimension | DailyMed Source Export | DailyMed Neo4j Load | Global Multi-Source Neo4j | DailyMed RDF Artifact (`/rwdkn`) |
| :--- | ---: | ---: | ---: | ---: |
| **Nodes** | 271 | 271 | 7,094 | 271 Entities |
| **Edges / Relationships** | 1,946 | 1,946 | 19,430 | 46,842 N-Triples (Reified) |
| **DrugClass Nodes** | 27 | 27 | 31 | 27 Typed Classes |
| **Subclass Relationships** | 42 | 42 | 49 | 42 Direct `rdfs:subClassOf` |

#### Subclass Cardinality Reconciliation (42 vs. 49 Edges)
* **36 Edges**: RxNav/RxClass pharmacologic class relationships (EPC, MoA, CS, ATC, Chemical/Ingredient) in DailyMed.
* **6 Edges**: FDA SPL direct pharmacologic class relationships extracted from DailyMed XMLs.
* **Total DailyMed Subclass Edges**: $36 + 6 = \mathbf{42}$.
* **7 Edges**: PubMed-specific drug subclass relationships.
* **Total Global Multi-Source Subclass Edges in Neo4j**: $42 + 7 = \mathbf{49}$.

---

## 3. Detailed Technical Critique of Review Issues

### ISSUE-1: RDF vs. LPG Projection Contracts
* **Review Finding**: Total graph count equality is invalid without explicit projection contracts.
* **Audit Verdict**: **Agreed (High Priority)**. LPG is a property graph projection optimized for graph algorithms and Cypher queries; RDF is an ontological projection optimized for formal semantics and SPARQL federation. The system must publish a formal dual-projection specification defining what is natively projected vs. what is linked.

### ISSUE-2 & ISSUE-3: DrugClass Hierarchy & Axis Boundaries
* **Review Finding**: Inconsistent `DrugClass` typing and subclass edge reconciliation.
* **Audit Verdict**: **Addressed in Epoch-3 Code & Reconciled**.
  1. Substance-grouping class axes (EPC, MoA, CS, EXT, Chemical/Ingredient) receive `schema:DrugClass` and dual W3C OWL 2 typing (`owl:Class` + `owl:NamedIndividual`).
  2. **Physiologic Effect (PE) Boundary**: PE is intentionally mapped to `biolink:PhysiologicalProcess` under `PharmacologicClassAxis`, not `schema:DrugClass`.
  3. Direct `rdfs:subClassOf` triples are materialized for all 42 DailyMed taxonomic relationships.

### ISSUE-4 to ISSUE-9: Clinical Text, Sections & Conditionality (CQ-1 to CQ-5)
* **Review Finding**: Original sentences and conditionality ("adjunct to diet and exercise") are stored in PostgreSQL audit tables but are not directly queryable in graph properties.
* **Audit Verdict**: **Agreed (Partial)**.
  * *Evidence Requirements by Edge Family*: `llm_assertion` edges require `has_evidence`, `sentence_id`, and `triple_id` links to `triple_provenance`. In contrast, `pharm_class_edge` and `metadata_publisher` carry source authority (`infores:dailymed`, `relation_source="RxClass"`) without clinical sentence IDs.
  * *Next Steps*: To enable purely graph-native execution of CQ-1 (wording diffs) and CQ-5 (conditionality), the extraction pipeline should materialize discrete qualifiers (`condition_qualifier`, `negated`, `source_section`) directly on edge properties.

### ISSUE-10: Date Scope (`effective_time` vs. `dcterms:issued`)
* **Review Finding**: Date fields represent label-document publication timing, not ingredient marketing windows.
* **Audit Verdict**: **Agreed & Formally Enforced**.
  * Codified in LinkML schema v0.7.0 (`effective_time` slot description: *"SPL document effective date for the label version (yyyymmdd literal). This is source-label provenance, not ingredient or RxCUI marketing availability"*).

### ISSUE-11: Insight Lookup Case Sensitivity
* **Review Finding**: `semaglutide` (lowercase) returned empty results while `Semaglutide` (title case) succeeded in the Insight explorer.
* **Audit Verdict**: **Resolved**. The resolver query in `subprojects/insight-studio/server/app.js` now uses `(toLower(s.name) = toLower($name) OR $name IN s.synonym OR s.id = $name)` to support lowercase, title-case, uppercase, and brand synonym matching.

### ISSUE-12 & ISSUE-13: Release Manifests & Storage Controls
* **Review Finding**: Manifests contained development `localhost` URLs and lacked immutable archive hashes.
* **Audit Verdict**: **Partially Addressed Locally; Open for Release**. Source export manifests now declare `kgx_export_epoch: 3` and `model_version: "0.7.0"`. Public participant and release endpoints should parameterize URLs to `https://rwdkn.kolac.us` and attach SHA-256 release checksums.

---

## 4. Architectural Capability Matrix

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ARCHITECTURAL CAPABILITY                           │
├──────────────────────────────┬─────────────────────────┬────────────────────┤
│ Capability                   │ Verified Status         │ Validation Base    │
├──────────────────────────────┼─────────────────────────┼────────────────────┤
│ 23-Column KGX TSV Wire Format│ IMPLEMENTED / TESTED    │ LinkML 0.7.0 / 289T│
│ Neo4j DailyMed Load          │ LOADED / SOURCE-VERIFIED│ 271 Nodes, 1,946 E │
│ Neo4j Global Multi-Source    │ LOADED / LIVE           │ 7,094 N, 19,430 E  │
│ W3C OWL 2 RDF Generator      │ IMPLEMENTED / TESTED    │ 46,842 N-Triples   │
│ Fuseki /rwdkn SPARQL Serving │ DEPLOYED / QUERYABLE    │ Live SPARQL Tested │
│ Evidence Bedrock             │ LOADED / AUDIT-READY    │ PG 18.4k Triples   │
│ Insight Case-Insensitive UI  │ IMPLEMENTED             │ app.js Updated     │
│ Immutable Release Packaging  │ PLANNED (Future)        │ Manifest SHA-256   │
└──────────────────────────────┴─────────────────────────┴────────────────────┘
```
