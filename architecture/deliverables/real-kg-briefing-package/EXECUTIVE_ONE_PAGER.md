# REAL-KG · Executive One-Pager

**Organization:** NIH / NLM / LHNCBC · Applied Clinical Informatics Branch  
**Prepared By:** Chandra Kola  
**Target:** NLM Leadership, Clinical Officers, and K3 Engineering Team  
**Date:** October 13, 2026  

---

### What is REAL-KG?
**REAL-KG** is NLM's pharmaceutical knowledge network designed to bridge **regulatory drug label claims (DailyMed)**, **peer-reviewed scientific literature (PubMed)**, **post-marketing spontaneous reports (FAERS)**, and **real-world longitudinal patient data (CPRD Aurum)** into an explainable, auditable knowledge graph.

---

### Key Technical Pillars

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ 1. HYBRID BioNLP PIPELINE                                               │
  │    • pySBD: Biomedical sentence segmentation (protects abbreviations,   │
  │      dosages, p-values).                                                │
  │    • scispaCy & negspacy: Biomedical NER & clinical negation detection. │
  │    • rapidfuzz: Grounding check (>=80%) to prevent hallucinations.     │
  │    • 4-Tier Normalization: Local UMLS (105.8M rows), RxNav, SapBERT,    │
  │      and NLM UTS REST API.                                              │
  ├─────────────────────────────────────────────────────────────────────────┤
  │ 2. DUAL SERVING GRAPH TOPOLOGY                                          │
  │    • Neo4j LPG: 7,094 nodes, 19,430 edges for multi-hop graph           │
  │      algorithms & Natural Language Querying (NLQ via Bedrock Claude).   │
  │    • Apache Jena Fuseki RDF: W3C OWL 2 Punning (owl:Class +             │
  │      owl:NamedIndividual) and reified triples for K3 / SPARQL.          │
  ├─────────────────────────────────────────────────────────────────────────┤
  │ 3. IMMUTABLE AUDIT BEDROCK (PostgreSQL lab_db)                          │
  │    • triples: 18,415 clinical SPO assertions with LinkML slots.         │
  │    • triple_provenance: 16,293 exact raw clinical sentences.           │
  │    • sentence_stage_events: 96,831 lifecycle events for full audit.     │
  ├─────────────────────────────────────────────────────────────────────────┤
  │ 4. UPDATE & GOVERNANCE CADENCE                                          │
  │    • DailyMed: Automated monthly batch runs on new FDA SPL releases.    │
  │    • PubMed: Incremental runs on monitored PMIDs.                       │
  │    • FAERS: Automated quarterly runs on FDA quarterly safety data.      │
  │    • CPRD Aurum: Governed strictly offline/ad hoc by Dr. Mehmet         │
  │      Kayaalp (PHI compliant; only aggregate profiles enter graph).      │
  │    • HITL: Triple Audit Explorer (/demo/triples/) allows SME            │
  │      informaticians to approve/quarantine candidate assertions.         │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

### Core Competency Questions Answered
* **CQ-1:** Discrepancy analysis between FDA label package inserts and newly published PubMed clinical trial results.
* **CQ-2 & CQ-3:** Pharmacologic class hierarchy traversals (MoA, EPC, PE, ATC) and class-wide adverse effect profiling (e.g. GLP-1 receptor agonists).
* **CQ-4:** Contrasting real-world clinical incidence rates (CPRD) against spontaneous reporting rates (FAERS) and label warnings.
* **CQ-5:** Extracting and querying discrete conditionality (e.g., *"indicated as an adjunct to diet and exercise in adults with Type 2 Diabetes"* vs. monotherapy).
