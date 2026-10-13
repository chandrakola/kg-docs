# Ready-to-Send Email Reply Draft

**To:** Kathy, K3 Vendor Engineering Team  
**Cc:** Mehmet Kayaalp, REAL-KG / NLM Core Team  
**From:** Chandra Kola (Applied Clinical Informatics Branch, LHNCBC / NLM / NIH)  
**Subject:** Re: REAL-KG: Data Sources, NLP Transformation, RDF/LPG Ingestion & Update Process  
**Attachments:** `real-kg-architecture-and-nlp-briefing.html` *(and/or `REAL-KG_Architecture_and_BioNLP_Briefing_Package.zip`)*  

---

Hi Kathy, K3 Team, and Mehmet,

Following up on Mehmet’s note regarding how **DailyMed**, **PubMed**, and **FAERS** are transformed, imported, and updated in REAL-KG:

To make our architecture, NLP approach, and RDF integration as clear as possible ahead of our core meeting, I have prepared a comprehensive, interactive technical briefing document: **[REAL-KG Architecture & BioNLP Briefing]** (attached as a self-contained HTML file).

Here is the high-level summary addressing the team's questions:

### 1. How Sources are Transformed (Hybrid BioNLP & LLM Engine)
* **Structure & Parsing (`source-nexus`)**: DailyMed SPL XMLs are parsed by LOINC section codes (e.g., Boxed Warning `34066-1`, Indications `34067-9`, Adverse Reactions `34084-4`). PubMed XMLs prioritize structured *Results* and *Conclusions*.
* **Extraction (`assertion-engine`)**: We combine classical BioNLP (`pySBD` biomedical sentence segmenter, `scispaCy` entity recognition, `negspacy` clinical negation, and `rapidfuzz` evidence grounding) with structured LLM extraction (Google Gemini Batch / AWS Bedrock Claude) enforced by strict JSON schemas. Every claim is anchored to verbatim source sentences in our PostgreSQL audit bedrock.
* **Multi-Tier Normalization (`concept-align`)**: Concepts are normalized deterministically across RxNorm, MED-RT, MedDRA, SNOMED-CT, MeSH, and MONDO using a tiered hierarchy: Tier-0 local UMLS Metathesaurus (105M rows), Tier-1 local RxNav, Tier-2 Cambridge SapBERT dense embeddings, and remote UTS.
* **FAERS Signals**: Ingested as spontaneous safety signals with report count qualifiers, modeled as statistical associations (`biolink:associated_with`) to keep them distinct from FDA label boxed warnings.

### 2. How Sources are Imported (Dual-Graph Serving & RDF Parity)
* **K3 / RDF Serving**: We project canonical W3C OWL 2 RDF (N-Triples/Turtle) for SPARQL endpoints (Apache Jena Fuseki). We implement **W3C OWL 2 Punning** (`owl:Class` + `owl:NamedIndividual`) so drug classes function both as semantic types and graph entities, with direct `rdfs:subClassOf` and reified statements for evidence provenance.
* **Property Graph Serving**: Ingested into Neo4j LPG for multi-hop graph algorithms and natural language querying (NLQ via Bedrock Claude) in Insight Studio.

### 3. Update Cadence & Human-in-the-Loop (HITL) Validation
* **Cadence**: DailyMed runs periodically against FDA SPL release updates; PubMed runs incrementally against monitored bibliographies; FAERS runs quarterly on new FDA quarterly releases. (CPRD is maintained ad hoc/offline by Mehmet due to PHI governance).
* **Quality Gates & SME Review**: In addition to automated fail-closed contract validators (`validate_contracts.py`), newly extracted triples and quarantined discrepancies are reviewed by clinicians/informaticians using our **Triple Audit Explorer** (`/demo/triples/`) before being tagged and promoted to the live graphs.

---

Please open the attached **`real-kg-architecture-and-nlp-briefing.html`** in any web browser to view the interactive architecture diagrams, BioNLP workflows, code samples, and CQ competency question mappings.

Looking forward to discussing this in detail during our next core meeting.

Best regards,  
**Chandra Kola**  
Lead / Applied Clinical Informatics  
LHNCBC / NLM / NIH  
