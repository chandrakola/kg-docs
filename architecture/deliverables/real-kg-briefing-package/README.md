# REAL-KG Technical Briefing & Architecture Deliverable Package

**Package Identifier:** `REAL-KG-DELIVERABLE-PKG-2026-10-13`  
**Organization:** National Institutes of Health (NIH) / National Library of Medicine (NLM) / LHNCBC  
**Branch:** Applied Clinical Informatics Branch  
**Prepared By:** Chandra Kola  
**Delivered To:** Dr. Mehmet Kayaalp, Kathy, NLM Core Team, K3 Vendor Engineering Team  
**Date:** October 13, 2026  

---

## 1. Package Purpose & Overview

This deliverable package provides the official technical briefing for the **REAL-KG (Real-World Evidence & Adverse Event Knowledge Graph)** platform. It addresses the questions raised regarding:
1. **Data Sources & Scope**: The four primary evidence tiers (DailyMed, PubMed, FAERS, CPRD Aurum).
2. **Hybrid BioNLP Transformation**: Sentence boundary disambiguation (`pySBD`), entity recognition (`scispaCy`), clinical negation (`negspacy`), fuzzy evidence grounding (`rapidfuzz`), and multi-tier UMLS/RxNav normalization.
3. **LLM Orchestration & Safety**: Structured JSON schema extractions, Gemini Batch API quota coordination, and persistent disk caching.
4. **Dual-Graph Serving & K3 RDF Standards**: Property Graph (Neo4j LPG) for multi-hop graph algorithms and NLQ, alongside canonical Semantic Web (W3C RDF/SPARQL) projections using **W3C OWL 2 Punning** (`owl:Class` + `owl:NamedIndividual`).
5. **Update Cadence & Human-in-the-Loop Governance**: Automated batch release cycles, quality gates, and SME review surfaces.

---

## 2. Package Contents & Files

| File Name | Format | Description & Usage |
| :--- | :--- | :--- |
| **`real-kg-architecture-and-nlp-briefing.html`** | Standalone HTML | **Primary Deliverable.** Interactive web portal featuring responsive SVG architecture diagrams, clickable 5-stage pipeline stepper, interactive BioNLP tabs, and dark/light themes. Open in any modern web browser. |
| **`real-kg-architecture-and-nlp-briefing.md`** | Markdown | Complete technical briefing formatted in GitHub Flavored Markdown for offline reading, terminal review, or repository check-in. |
| **`EXECUTIVE_ONE_PAGER.md`** | Markdown / Text | High-level executive summary tailored for leadership and clinical officers. |
| **`EMAIL_REPLY_DRAFT.md`** | Markdown / Text | Ready-to-send email response to Kathy, Dr. Mehmet Kayaalp, and the K3 team. |
| **`checksums.sha256`** | SHA-256 Digest | Cryptographic checksums ensuring file integrity across transfers. |

---

## 3. How to Use & Open the Primary Deliverable

### Viewing the Interactive HTML Portal
The file `real-kg-architecture-and-nlp-briefing.html` is completely self-contained (all styles, scripts, and vector diagrams are embedded; no external server or build step required).

* **On Linux Desktop**:
  ```bash
  xdg-open real-kg-architecture-and-nlp-briefing.html
  ```
* **On macOS**:
  ```bash
  open real-kg-architecture-and-nlp-briefing.html
  ```
* **On Windows**:
  ```cmd
  start real-kg-architecture-and-nlp-briefing.html
  ```
* **Direct File Open**: Simply double-click the file in your file explorer, or drag and drop it into Google Chrome, Microsoft Edge, Mozilla Firefox, or Apple Safari.
* **Exporting to PDF**: Open the HTML file and click the **"Export PDF"** button in the top navigation bar, or press `Ctrl+P` / `Cmd+P`.

---

## 4. Summary of Key Answers for K3 & NLM Team

1. **How DailyMed, PubMed, and FAERS are Transformed & Imported**:
   - Transformed via a **Hybrid BioNLP pipeline** combining `pySBD` sentence segmentation, `scispaCy` entity extraction, `negspacy` negation detection, and structured LLM relation extraction governed by strict JSON schemas.
   - Grounded with `rapidfuzz` ($\ge 80\%$) into PostgreSQL (`triple_provenance`, `triples`).
   - Normalized across RxNorm, MED-RT, MedDRA, SNOMED-CT, MeSH, and MONDO.
   - Dual-imported into **Neo4j** (property graph) and **Apache Jena Fuseki** (W3C OWL 2 RDF).
2. **Update Process**:
   - **DailyMed**: Automated batch runs periodically against updated FDA SPL release archives.
   - **PubMed**: Incremental batch runs triggered against monitored literature bibliographies.
   - **FAERS**: Automated quarterly runs aligned with FDA quarterly data dumps.
   - **CPRD**: Governed strictly offline/ad hoc by Dr. Mehmet Kayaalp due to PHI protection.
   - **Human-in-the-Loop (HITL)**: Subject Matter Experts validate candidate assertions in the **Triple Audit Explorer** (`/demo/triples/`) before promotion to live serving databases.
3. **K3 RDF Integration**:
   - Drug classes use **W3C OWL 2 Punning** (`owl:Class` + `owl:NamedIndividual`) with direct `rdfs:subClassOf` materialization and reified statement triples for evidence provenance, ensuring 100% compatibility with SPARQL endpoints and RDF reasoners.
