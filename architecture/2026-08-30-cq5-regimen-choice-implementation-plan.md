# Implementation Plan: Standards-Aligned Regimen Choice Architecture for CQ-5 (ISSUE-9)

**Document ID:** `RWDKN-PLAN-2026-08-30-CQ5-REV3`  
**Date:** August 30, 2026  
**Status:** PROPOSED FOR REVIEW (Definitive Technical Specification Resolving F-01 through F-16)  
**Supersedes:** `RWDKN-PLAN-2026-08-30-CQ5-REV2`  
**Related Documents:**  
* Reviewer Feedback: [`2026-08-30-cq5-regimen-choice-group-feedback.md`](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/2026-08-30-cq5-regimen-choice-group-feedback.md)  
* Technical Review 1: [`2026-08-30-cq5-regimen-choice-implementation-plan-review.md`](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/2026-08-30-cq5-regimen-choice-implementation-plan-review.md)  
* Technical Review 2 (REV2 Review): [`rwdkn-data-pipeline/docs/2026-08-30-cq5-regimen-choice-implementation-plan-review.md`](file:///home/chandrakola/development/NLM/kg/rwdkn-data-pipeline/docs/2026-08-30-cq5-regimen-choice-implementation-plan-review.md)  
* Target Repositories: `rwdkn-data-pipeline`, `rwdkn-service`, `kg-docs`  

---

## 1. Executive Summary & Core Semantic Invariants

### 1.1. Purpose & Clinical Scope
This document specifies the normative implementation plan for resolving **ISSUE-9 / CQ-5 (Conditional & Adjunct Indications)** across the Real World Drug Knowledge Network (RWDKN). It models complex clinical conditionality (e.g. *"indicated as an adjunct to diet and exercise to improve glycemic control in adults with type 2 diabetes mellitus"*) using a standards-aligned, compositional **`RegimenChoiceGroup` + `RegimenOption`** model.

### 1.2. Non-Negotiable Semantic Invariants
1. **Open-World Semantics (Absence $\ne$ Monotherapy)**:
   * A treatment assertion lacking a regimen choice group means **regimen unspecified**, *never* monotherapy.
   * Monotherapy must be explicit, positive, and backed by source evidence (`ExplicitMonotherapy`).
2. **Indivisible Option Bundles**:
   * Permitted therapy combinations (e.g. `(Dietary treatment + Exercise therapy)` or `(Drug X + Metformin)`) are encapsulated as distinct `RegimenOption` units. They are never flattened into a single concatenated list to prevent invalid cross-combination synthesis.
3. **Exact Canonical FHIR R5 Vocabularies**:
   * Selection behavior: `http://hl7.org/fhir/action-selection-behavior` (`exactly-one`, `all`, `all-or-none`, `at-most-one`, `one-or-more`, `any`)
   * Therapy relationship: `http://hl7.org/fhir/therapy-relationship-type` (`indicated-only-with`, `indicated-except-with`, `contraindicated-only-with`, `contraindicated-except-with`, `replace-other-therapy`, etc.)
4. **Collision-Proof Deterministic Identity (UUIDv5)**:
   * Generated via canonical JSON byte strings using the immutable RWDKN namespace UUID.
5. **Non-Lossy Terminology Quarantine**:
   * If a concomitant therapy term cannot be grounded to a standard CURIE, the **base treatment association is accepted**, while the regimen modifier is quarantined in `regimens.jsonl` for clinical review.
6. **Zero Drift on Closed TSV Contracts**:
   * The physical KGX TSV layer remains strictly locked to the 20-column `nodes.tsv` and 23-column `edges.tsv` profile. Regimens are serialized in a versioned, manifest-bound companion artifact: `regimens.jsonl`.

---

## 2. Terminology & Standards Binding Specification

### 2.1. Pinned Terminology Releases
* **SNOMED CT**: US Edition, March 2026 Release (`http://snomed.info/sct/731000124108/version/20260301`)
* **RxNorm**: Pinned NLM RxNorm Monthly Release (`http://purl.bioontology.org/ontology/RXNORM/`)
* **HL7 FHIR**: R5 Core Specification (`http://hl7.org/fhir/R5/`)
* **Biolink Model**: Pinned version 4.3.6 (`https://w3id.org/biolink/biolink-model/`)

### 2.2. Authoritative Terminology Binding Decision Table

| Concept Key | Canonical CURIE | Preferred Label (Pinned Release) | Code System | Semantic Category | Application & Extraction Rule |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `dietary_treatment` | `SNOMEDCT:284071006` | *Dietary treatment for disorder* | SNOMED CT (US) | `biolink:Procedure` | Matched for general diet, dietary modification, or caloric control phrases |
| `exercise_therapy` | `SNOMEDCT:229065009` | *Exercise therapy* | SNOMED CT (US) | `biolink:Procedure` | Matched for physical activity, exercise, or structured fitness interventions |
| `lifestyle_modification` | `SNOMEDCT:429300007` | *Lifestyle modification* | SNOMED CT (US) | `biolink:Procedure` | Matched for combined lifestyle / behavioral modification statements |
| `active_drug_substance` | `RXCUI:<id>` | *Ingredient Name* | RxNorm (IN/PIN) | `biolink:Drug` | Pinned active pharmaceutical ingredient |
| `selection_behavior` | `fhir-select:<code-id>` | *FHIR Action Selection* | HL7 FHIR R5 | Value Set Code | `exactly-one`, `all`, `all-or-none`, `at-most-one`, `one-or-more`, `any` |
| `therapy_relationship` | `fhir-rel:<code-id>` | *FHIR Therapy Relationship* | HL7 FHIR R5 | Value Set Code | `indicated-only-with`, `indicated-except-with`, `contraindicated-only-with`, etc. |

---

## 3. Deterministic Identity Specification (UUIDv5)

All choice-group and option identifiers are generated using **UUIDv5** hashing over UTF-8 canonical JSON bytes (keys sorted alphabetically, no whitespace, list elements sorted deterministically).

### 3.1. Immutable RWDKN Namespace
```python
import uuid

# Immutable Namespace UUID for RWDKN (UUIDv5 of DNS namespace on "realkg.nlm.nih.gov")
RWDKN_NAMESPACE_UUID = uuid.UUID("645391e8-7828-5690-b1fa-8a603c7e7dd6")
```

### 3.2. Choice Group ID Recipe
```python
def make_choice_group_id(association_id: str, selection_behavior: str, therapy_relationship: str) -> str:
    canonical_payload = {
        "association_id": association_id,
        "selection_behavior": selection_behavior,
        "therapy_relationship": therapy_relationship,
    }
    canonical_str = json.dumps(canonical_payload, sort_keys=True, separators=(",", ":"))
    return f"urn:uuid:{uuid.uuid5(RWDKN_NAMESPACE_UUID, canonical_str)}"
```

### 3.3. Regimen Option ID Recipe
```python
def make_regimen_option_id(
    choice_group_id: str,
    regimen_type: str,
    primary_therapy: str,
    concomitant_therapies: list[str],
) -> str:
    canonical_payload = {
        "choice_group_id": choice_group_id,
        "concomitant_therapies": sorted(set(concomitant_therapies)),
        "primary_therapy": primary_therapy,
        "regimen_type": regimen_type,
    }
    canonical_str = json.dumps(canonical_payload, sort_keys=True, separators=(",", ":"))
    return f"urn:uuid:{uuid.uuid5(RWDKN_NAMESPACE_UUID, canonical_str)}"
```

---

## 4. Physical Wire Contract: Manifest-Bound Companion Sidecar (`regimens.jsonl`)

The export layout preserves existing 20-column `nodes.tsv` and 23-column `edges.tsv` while providing rich nested regimen structures in `regimens.jsonl`:

```text
application/kg-data/kgx/dailymed/
├── nodes.tsv                 # Standard 20-column KGX Nodes (includes Therapy context nodes)
├── edges.tsv                 # Standard 23-column KGX Edges (TREATS edge with edge.id)
├── regimens.jsonl            # Structured RegimenRecord JSON lines (Accepted & Quarantined)
└── export_manifest.json      # Checksums, sizes, and record counts for all 3 artifacts
```

### 4.1. `RegimenRecord` JSON Schema Specification

#### Branch A: Accepted Regimen Record (`status: "accepted"`)
```json
{
  "schema_version": "1.0.0",
  "association_id": "urn:uuid:6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "therapy_relationship": "indicated-only-with",
  "status": "accepted",
  "evidence_sentence_id": "s-ozempic-ind-01",
  "raw_evidence_text": "as an adjunct to diet and exercise to improve glycemic control in adults with type 2 diabetes mellitus",
  "regimen_group": {
    "id": "urn:uuid:a1b2c3d4-e5f6-5a7b-8c9d-0e1f2a3b4c5d",
    "selection_behavior": "all",
    "options": [
      {
        "id": "urn:uuid:b2c3d4e5-f6a7-5b8c-9d0e-1f2a3b4c5d6e",
        "regimen_type": "combination",
        "primary_therapy": "RXCUI:1991302",
        "concomitant_therapies": [
          {
            "curie": "SNOMEDCT:284071006",
            "name": "Dietary treatment for disorder",
            "raw_text": "diet"
          },
          {
            "curie": "SNOMEDCT:229065009",
            "name": "Exercise therapy",
            "raw_text": "exercise"
          }
        ]
      }
    ]
  }
}
```

#### Branch B: Quarantined Regimen Record (`status: "quarantined"`)
```json
{
  "schema_version": "1.0.0",
  "association_id": "urn:uuid:8fa6c721-1eb4-4c8d-90c1-34e89f01a2b3",
  "therapy_relationship": "indicated-only-with",
  "status": "quarantined",
  "quarantine_reason": "unmapped_concomitant_phrase",
  "evidence_sentence_id": "s-invokana-ind-02",
  "raw_evidence_text": "as an adjunct to standard diabetic care including behavioral coaching",
  "unresolved_terms": ["behavioral coaching"],
  "regimen_group": null
}
```

---

## 5. Multi-Phase Implementation Plan

### Phase 0: Schema, LinkML & Generated Contract Infrastructure
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/biolink-bridge/`)

1. **Update LinkML Model (`subprojects/biolink-bridge/schema/rwdkn-model.yaml`)**:
   * Add `RegimenRecord` class modeling the physical sidecar line.
   * Add `RegimenChoiceGroup`, `RegimenOption`, and `ProhibitedRegimen` semantic classes.
   * Add `ActionSelectionBehaviorEnum` with canonical FHIR R5 values (`all`, `any`, `all-or-none`, `exactly-one`, `at-most-one`, `one-or-more`).
   * Add `TherapyRelationshipEnum` with canonical FHIR R5 values (`indicated-only-with`, `indicated-except-with`, `contraindicated-only-with`, etc.).
   * Add `RegimenTypeEnum` with values (`combination`, `monotherapy`).
   * Add `RegimenStatusEnum` with values (`accepted`, `quarantined`).
2. **Update Contract Generator (`gen_contracts.py`)**:
   * Generate `regimen_record.schema.json` alongside `node_record.schema.json` and `association_record.schema.json`.
   * Include `regimen_record.schema.json` in `resources/generated/MANIFEST.json`.
3. **Register SHACL Overlay (`subprojects/biolink-bridge/resources/rwdkn-regimen-shapes.ttl`)**:
   * Formally bind `rwdkn-regimen-shapes.ttl` in generated contracts, checksummed in `MANIFEST.json`.
   * Enforce SHACL constraints: options require `primaryTherapy`; `combination` options require $\ge 1$ concomitant therapy; `monotherapy` options require 0 concomitant therapies.

---

### Phase 1: Stage 2 Extraction, Concept Normalization & Context Node Lifecycle
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/assertion-engine/`, `subprojects/concept-align/`)

1. **Assertion Extraction (`subprojects/assertion-engine/`)**:
   * Output structured regimen assertions alongside SPO triples with exact sentence span bindings.
   * Inherit `primary_therapy` deterministically from the triple subject CURIE (`RXCUI`).
2. **Concept Normalization & Non-Lossy Quarantine (`subprojects/concept-align/`)**:
   * Map concomitant therapies using the Authoritative Terminology Binding Decision Table (Section 2.2).
   * Emit therapy context nodes (`SNOMEDCT:284071006`, `SNOMEDCT:229065009`, `SNOMEDCT:429300007`) as first-class rows in `nodes.tsv` with category `biolink:Procedure` and `node_type: "TherapyContext"`.
   * If a concomitant therapy cannot be grounded, emit a `status: "quarantined"` record into `regimens.jsonl` while accepting and emitting the base treatment edge into `edges.tsv`.

---

### Phase 2: Pipeline Export & Quality Guardian Gates
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/biolink-bridge/`, `subprojects/quality-guardian/`)

1. **Export Engine (`export_kgx.py`)**:
   * Emit `regimens.jsonl` alongside `nodes.tsv` and `edges.tsv`.
   * Compute SHA-256, byte size, and row/record count for all 3 artifacts in `export_manifest.json`:
     ```json
     {
       "nodes_written": 182,
       "nodes_sha256": "...",
       "edges_written": 245,
       "edges_sha256": "...",
       "regimens_written": 14,
       "regimens_accepted": 12,
       "regimens_quarantined": 2,
       "regimens_sha256": "..."
     }
     ```
2. **Quality Guardian Validation Gate (`validate_stage4_ontology.py`)**:
   * Validate `regimens.jsonl` against `regimen_record.schema.json`.
   * Verify total invariant count: `total_candidates == accepted + quarantined + rejected`.
   * Validate that all accepted therapy CURIEs match approved vocabularies and that UUIDv5 rules are obeyed.

---

### Phase 3: Dual Serving Projections (Neo4j LPG + Apache Jena Fuseki RDF)
**Target Repositories:** `rwdkn-data-pipeline`, `rwdkn-service`

1. **Consolidated Neo4j Loader (`load_kgx_to_neo4j.py` & `rwdkn-service/.../neo4j_loader.py`)**:
   * Materialize reified graph topology for accepted records:
     ```cypher
     (:Drug {id: 'RXCUI:1991302'})-[:TREATS {id: 'urn:uuid:assoc-101'}]->(:Disease {id: 'SNOMEDCT:44054006'})
     (:Association {id: 'urn:uuid:assoc-101'})-[:HAS_REGIMEN_CHOICE]->(:RegimenChoiceGroup {
         id: 'urn:uuid:rcg-01',
         selection_behavior: 'all',
         therapy_relationship: 'indicated-only-with'
     })
     (:RegimenChoiceGroup {id: 'urn:uuid:rcg-01'})-[:HAS_OPTION]->(:RegimenOption {
         id: 'urn:uuid:ro-01',
         regimen_type: 'combination',
         primary_therapy: 'RXCUI:1991302'
     })
     (:RegimenOption {id: 'urn:uuid:ro-01'})-[:PRIMARY_THERAPY]->(:Drug {id: 'RXCUI:1991302'})
     (:RegimenOption {id: 'urn:uuid:ro-01'})-[:CONCOMITANT_THERAPY]->(:Therapy {
         id: 'SNOMEDCT:284071006',
         name: 'Dietary treatment for disorder',
         category: 'biolink:Procedure'
     })
     (:RegimenOption {id: 'urn:uuid:ro-01'})-[:CONCOMITANT_THERAPY]->(:Therapy {
         id: 'SNOMEDCT:229065009',
         name: 'Exercise therapy',
         category: 'biolink:Procedure'
     })
     ```
   * Add uniqueness constraints on `(Association.id)`, `(RegimenChoiceGroup.id)`, `(RegimenOption.id)`, and `(Therapy.id)`.

2. **RDF / Fuseki Projection (`rdf_projection.py` & `rwdkn-service`)**:
   * Project reified Turtle / N-Triples:
     ```turtle
     @prefix biolink:     <https://w3id.org/biolink/vocab/> .
     @prefix fhir-select: <http://hl7.org/fhir/action-selection-behavior/> .
     @prefix fhir-rel:    <http://hl7.org/fhir/therapy-relationship-type/> .
     @prefix rwdkn:       <https://w3id.org/realkg/> .

     <urn:uuid:assoc-101> a biolink:Association ;
         biolink:subject <https://mor.nlm.nih.gov/RxNav/search?searchBy=RXCUI&searchTerm=1991302> ;
         biolink:predicate biolink:treats ;
         biolink:object <http://snomed.info/id/44054006> ;
         rwdkn:permittedRegimenGroup <urn:uuid:rcg-01> .

     <urn:uuid:rcg-01> a rwdkn:RegimenChoiceGroup ;
         rwdkn:selectionBehavior fhir-select:all ;
         rwdkn:therapyRelationship fhir-rel:indicated-only-with ;
         rwdkn:regimenOption <urn:uuid:ro-01> .

     <urn:uuid:ro-01> a rwdkn:RegimenOption ;
         rwdkn:regimenType rwdkn:CombinationRegimen ;
         rwdkn:primaryTherapy <https://mor.nlm.nih.gov/RxNav/search?searchBy=RXCUI&searchTerm=1991302> ;
         rwdkn:concomitantTherapy <http://snomed.info/id/284071006> ,
                                  <http://snomed.info/id/229065009> .
     ```
   * Update `rwdkn-service/scripts/install_release.py` to perform atomic dataset publication in Apache Jena Fuseki with release checksum verification.

---

### Phase 4: Automated CQ-5 Option-Level Dual-Graph Parity Gate
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/biolink-bridge/tests/`)

Implement `test_cq5_dual_store_parity.py` comparing canonical option-level tuples.

#### 4.1. Option-Level Canonical Parity Data Structure
```python
@dataclass(frozen=True)
class RegimenOptionParityTuple:
    association_id: str
    drug_curie: str
    indication_curie: str
    choice_group_id: str
    selection_behavior: str      # e.g. "all", "exactly-one"
    therapy_relationship: str    # e.g. "indicated-only-with"
    regimen_option_id: str
    regimen_type: str            # "combination" | "monotherapy"
    primary_therapy_curie: str   # e.g. "RXCUI:1991302"
    concomitant_therapies: tuple[str, ...]  # Alphabetically sorted tuple of CURIEs
```

#### 4.2. Exact Cypher Parity Query
```cypher
MATCH (d:Drug)-[r:TREATS]->(dis:Disease)
MATCH (assoc:Association {id: r.id})-[:HAS_REGIMEN_CHOICE]->(rcg:RegimenChoiceGroup)
MATCH (rcg)-[:HAS_OPTION]->(opt:RegimenOption)
OPTIONAL MATCH (opt)-[:CONCOMITANT_THERAPY]->(t:Therapy)
WITH r.id AS association_id,
     d.id AS drug_curie,
     dis.id AS indication_curie,
     rcg.id AS choice_group_id,
     rcg.selection_behavior AS selection_behavior,
     rcg.therapy_relationship AS therapy_relationship,
     opt.id AS regimen_option_id,
     opt.regimen_type AS regimen_type,
     opt.primary_therapy AS primary_therapy_curie,
     t.id AS therapy_curie
ORDER BY therapy_curie
RETURN association_id, drug_curie, indication_curie, choice_group_id,
       selection_behavior, therapy_relationship, regimen_option_id,
       regimen_type, primary_therapy_curie,
       collect(DISTINCT therapy_curie) AS concomitant_therapies;
```

#### 4.3. Exact SPARQL Parity Query
```sparql
PREFIX biolink:     <https://w3id.org/biolink/vocab/>
PREFIX fhir-select: <http://hl7.org/fhir/action-selection-behavior/>
PREFIX fhir-rel:    <http://hl7.org/fhir/therapy-relationship-type/>
PREFIX rwdkn:       <https://w3id.org/realkg/>

SELECT ?association_id ?drug_iri ?indication_iri ?choice_group_id
       ?selection_behavior_iri ?therapy_rel_iri ?regimen_option_id
       ?regimen_type_iri ?primary_therapy_iri
       (GROUP_CONCAT(DISTINCT STR(?therapy_iri); separator="|") AS ?concomitant_therapies_concat)
WHERE {
    ?assoc a biolink:Association ;
           biolink:subject ?drug_iri ;
           biolink:predicate biolink:treats ;
           biolink:object ?indication_iri ;
           rwdkn:permittedRegimenGroup ?rcg .
    
    ?rcg rwdkn:selectionBehavior ?selection_behavior_iri ;
         rwdkn:therapyRelationship ?therapy_rel_iri ;
         rwdkn:regimenOption ?opt .
    
    ?opt rwdkn:regimenType ?regimen_type_iri ;
         rwdkn:primaryTherapy ?primary_therapy_iri .
    
    OPTIONAL {
        ?opt rwdkn:concomitantTherapy ?therapy_iri .
    }
    BIND(STR(?assoc) AS ?association_id)
    BIND(STR(?rcg) AS ?choice_group_id)
    BIND(STR(?opt) AS ?regimen_option_id)
}
GROUP BY ?association_id ?drug_iri ?indication_iri ?choice_group_id
         ?selection_behavior_iri ?therapy_rel_iri ?regimen_option_id
         ?regimen_type_iri ?primary_therapy_iri
```

#### 4.4. Explicit Value Normalization Functions
```python
def normalize_fhir_select(val: str) -> str:
    return val.rsplit("/", 1)[-1].replace("fhir-select:", "")

def normalize_fhir_rel(val: str) -> str:
    return val.rsplit("/", 1)[-1].replace("fhir-rel:", "")

def normalize_regimen_type(val: str) -> str:
    name = val.rsplit("/", 1)[-1].replace("rwdkn:", "")
    if name in ("CombinationRegimen", "combination"):
        return "combination"
    if name in ("ExplicitMonotherapy", "monotherapy"):
        return "monotherapy"
    return name

def normalize_curie(iri_or_curie: str) -> str:
    if "RXCUI&searchTerm=" in iri_or_curie:
        return "RXCUI:" + iri_or_curie.split("searchTerm=")[-1]
    if "snomed.info/id/" in iri_or_curie:
        return "SNOMEDCT:" + iri_or_curie.split("snomed.info/id/")[-1]
    return iri_or_curie

def parse_concomitants(raw: Any) -> tuple[str, ...]:
    if isinstance(raw, list):
        return tuple(sorted({normalize_curie(c) for c in raw if c}))
    if isinstance(raw, str):
        if not raw.strip():
            return ()
        return tuple(sorted({normalize_curie(c) for c in raw.split("|") if c.strip()}))
    return ()
```

---

## 6. Multi-Scenario Clinical Gold Test Matrix

Acceptance testing includes an explicit 8-scenario versioned gold test suite (`tests/fixtures/regimen_gold_set.json`):

| Scenario ID | Test Case Category | Clinical Description / Input Text | Expected Action Selection | Expected Regimen Type | Expected Concomitant CURIEs | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GS-01` | Positive Adjunct Pair | *"as an adjunct to diet and exercise to improve glycemic control in T2D"* | `all` | `combination` | `["SNOMEDCT:229065009", "SNOMEDCT:284071006"]` | `accepted` |
| `GS-02` | Explicit Monotherapy | *"indicated as monotherapy for the treatment of hypertension"* | `exactly-one` | `monotherapy` | `[]` | `accepted` |
| `GS-03` | Unspecified (Absence) | *"indicated for the treatment of major depressive disorder"* | `N/A` (No record emitted) | `N/A` | `[]` | `N/A` |
| `GS-04` | Single Required Therapy | *"indicated in combination with metformin for glycemic control"* | `all` | `combination` | `["RXCUI:6809"]` (Metformin) | `accepted` |
| `GS-05` | Disjunctive Choice `(A+B) OR (C+D)` | *"indicated with diet and exercise, or with metformin and diet"* | `exactly-one` | 2 Options: `(Diet+Exercise)`, `(Metformin+Diet)` | Option 1 & Option 2 sets | `accepted` |
| `GS-06` | Prohibited Combination | *"contraindicated in combination with GLP-1 receptor agonists"* | `N/A` (Prohibition record) | `N/A` | `["RXCUI:..."]` | `prohibited` |
| `GS-07` | Unresolved Phrase | *"as an adjunct to behavioral coaching"* | `N/A` | `N/A` | `[]` | `quarantined` |
| `GS-08` | Legacy Epoch-2 Record | Pre-epoch-3 DailyMed export without `regimens.jsonl` | `N/A` | `N/A` | `[]` | `unspecified` |

---

## 7. Migration, Rollback & Service Delivery Plan

1. **Epoch Versioning**:
   * Model version: `0.7.0`, `kgx_export_epoch: 3`.
   * Epoch 2 releases (legacy) load cleanly without error; treatment edges remain `unspecified`.
   * Epoch 3 releases require `regimens.jsonl` validation.
2. **Idempotent Rollback**:
   * Neo4j loader and Fuseki dataset installer support atomic wipe-and-reload per `knowledge_source`.
   * If `regimens.jsonl` verification fails, the load transaction is aborted without touching active graph data.
3. **Service API Endpoint**:
   * Add `/explorer/api/drug/{curie}/regimens` to `rwdkn-service` exposing structured regimen choice groups to frontend components without raw Cypher or SPARQL queries.

---

## 8. Verification Commands & Acceptance Criteria

```bash
# 1. Regenerate LinkML contracts, JSON schemas, and check zero drift
cd subprojects/biolink-bridge && uv run python gen_contracts.py && uv run python gen_contracts.py --check

# 2. Run SHACL structural validation over regimen shapes
uv run pytest tests/test_regimen_choice_shacl.py -v

# 3. Run unit tests on deterministic UUIDv5 identity, JSONL codec, and gold set
uv run pytest tests/test_regimen_codec.py -v

# 4. Run dual-store option-level CQ-5 parity integration gate
uv run pytest tests/test_cq5_dual_store_parity.py -v

# 5. Run Quality Guardian ontology gate checks
cd ../quality-guardian && uv run pytest tests/
```

### Acceptance Checklist
* [ ] `regimen_record.schema.json` generated and checksummed in `resources/generated/MANIFEST.json`.
* [ ] Pinned SNOMED CT preferred labels (`Dietary treatment for disorder`, `Exercise therapy`, `Lifestyle modification`) verified.
* [ ] UUIDv5 identity produces zero collisions across all 8 gold-set scenarios.
* [ ] Missing regimen data remains `unspecified` and is never converted to monotherapy.
* [ ] `regimens.jsonl` checksum, size, and counts recorded in `export_manifest.json`.
* [ ] Quarantined records preserve raw evidence text without dropping base treatment edges.
* [ ] Option-level Cypher and SPARQL parity test passes with 100% tuple equality on Semaglutide, Tirzepatide, and Liraglutide.
* [ ] ISSUE-9 is closed only upon full passing of the acceptance suite.
