# Implementation Plan: Standards-Aligned Regimen Choice Architecture for CQ-5 (ISSUE-9)

**Document ID:** `RWDKN-PLAN-2026-08-30-CQ5-REV4`  
**Date:** August 30, 2026  
**Status:** APPROVED FOR IMPLEMENTATION (Definitive Specification Resolving All REV3 Review Conditions)  
**Supersedes:** `RWDKN-PLAN-2026-08-30-CQ5-REV3`  
**Related Documents:**  
* Reviewer Feedback: [`2026-08-30-cq5-regimen-choice-group-feedback.md`](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/2026-08-30-cq5-regimen-choice-group-feedback.md)  
* Technical Review 1: [`2026-08-30-cq5-regimen-choice-implementation-plan-review.md`](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/2026-08-30-cq5-regimen-choice-implementation-plan-review.md)  
* Technical Review 2: [`rwdkn-data-pipeline/docs/2026-08-30-cq5-regimen-choice-implementation-plan-review.md`](file:///home/chandrakola/development/NLM/kg/rwdkn-data-pipeline/docs/2026-08-30-cq5-regimen-choice-implementation-plan-review.md)  
* Technical Review 3 (REV3 Review): [`rwdkn-data-pipeline/docs/2026-08-30-cq5-regimen-choice-implementation-plan-rev3-review.md`](file:///home/chandrakola/development/NLM/kg/rwdkn-data-pipeline/docs/2026-08-30-cq5-regimen-choice-implementation-plan-rev3-review.md)  
* Target Repositories: `rwdkn-data-pipeline`, `rwdkn-service`, `kg-docs`  

---

## 1. Executive Summary & Semantic Invariants

### 1.1. Scope & Objective
This specification defines the definitive implementation plan to resolve **ISSUE-9 / CQ-5 (Conditional & Adjunct Indications)** in the Real World Drug Knowledge Network (RWDKN). It models complex clinical conditionality (e.g. *"indicated as an adjunct to diet and exercise to improve glycemic control in adults with type 2 diabetes mellitus"*) using a standards-aligned, compositional **`RegimenChoiceGroup` + `RegimenOption`** architecture.

### 1.2. Non-Negotiable Semantic Invariants
1. **Open-World Semantics (Absence $\ne$ Monotherapy)**:
   * A treatment assertion lacking a regimen choice group means **regimen unspecified**, *never* monotherapy.
   * Monotherapy must be explicit, positive, and backed by source evidence (`ExplicitMonotherapy`).
2. **Indivisible Option Bundles**:
   * Permitted therapy combinations (e.g. `(Dietary treatment + Exercise therapy)`) are encapsulated as distinct `RegimenOption` units. They are never flattened into a concatenated list.
3. **Orthogonal Semantic Axes**:
   * **`record_status`**: `accepted | quarantined | rejected` (pipeline ingestion/validation state)
   * **`clinical_disposition`**: `permitted | contraindicated | not_recommended` (clinical meaning)
   * **`regimen_type`**: `combination | monotherapy` (compositional structure)
4. **Exact Canonical HL7 FHIR R5 Vocabularies**:
   * Selection behavior: `http://hl7.org/fhir/action-selection-behavior` (`exactly-one`, `all`, `all-or-none`, `at-most-one`, `one-or-more`, `any`)
   * Therapy relationship: `http://hl7.org/fhir/therapy-relationship-type` (`indicated-only-with`, `indicated-except-with`, `contraindicated-only-with`, `contraindicated-except-with`, `replace-other-therapy`, etc.)
5. **Mathematically Verified Deterministic Identity (UUIDv5)**:
   * Fixed immutable RWDKN namespace derived from `uuid5(NAMESPACE_DNS, "realkg.nlm.nih.gov")` = `f472c728-48ba-5c19-b566-cc6217064e33`.
6. **Non-Lossy Terminology Quarantine**:
   * If a concomitant therapy term cannot be grounded to a standard CURIE, the **base treatment association is accepted**, while the regimen modifier is quarantined in `regimens.jsonl` with preserved candidate structures for clinical review.
7. **Zero Drift on Closed TSV Contracts**:
   * The physical KGX TSV layer remains strictly locked to the 20-column `nodes.tsv` and 23-column `edges.tsv` profile. Regimens are serialized in a manifest-bound companion artifact: `regimens.jsonl` under `model_version: "0.8.0"` and `regimen_contract_version: "1.0.0"`.

---

## 2. Pinned Terminology & Standards Binding Table

### 2.1. Pinned Authority Releases
* **SNOMED CT**: US Edition, March 2026 Release (`http://snomed.info/sct/731000124108/version/20260301`)
* **RxNorm**: Monthly Release 2026-03 (`http://purl.bioontology.org/ontology/RXNORM/`)
* **HL7 FHIR**: R5 Core Specification v5.0.0 (`http://hl7.org/fhir/R5/`)
* **Biolink Model**: Pinned version 4.3.6 (`resources/biolink-model-4.3.6.yaml`, `sha256:7fc26cfeeb17828cef21e37c738185cad93f69135d135e70dc531ecc7f83138a`)

### 2.2. Authoritative Terminology Binding Decision Table

| Concept Key | Canonical CURIE | Preferred Label (Pinned SNOMED March 2026) | Code System | Category | Node Type | Application Rule |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `dietary_treatment` | `SNOMEDCT:284071006` | *Dietary treatment for disorder* | SNOMED CT (US) | `biolink:Procedure` | `TherapyContext` | Matched for diet, dietary modification, or caloric control |
| `exercise_therapy` | `SNOMEDCT:229065009` | *Exercise therapy* | SNOMED CT (US) | `biolink:Procedure` | `TherapyContext` | Matched for exercise or structured physical activity |
| `lifestyle_modification` | `SNOMEDCT:429300007` | *Lifestyle modification* | SNOMED CT (US) | `biolink:Procedure` | `TherapyContext` | Matched for combined lifestyle / behavioral interventions |
| `active_drug_substance` | `RXCUI:<id>` | *Ingredient Name* | RxNorm (IN/PIN) | `biolink:Drug` | `Drug` | Pinned active pharmaceutical ingredient |
| `selection_behavior` | `fhir-select:<code-id>` | *FHIR Action Selection* | HL7 FHIR R5 | Value Set Code | `N/A` | `exactly-one`, `all`, `all-or-none`, `at-most-one`, `one-or-more`, `any` |
| `therapy_relationship` | `fhir-rel:<code-id>` | *FHIR Therapy Relationship* | HL7 FHIR R5 | Value Set Code | `N/A` | `indicated-only-with`, `contraindicated-only-with`, etc. |

---

## 3. Deterministic Identity Specification (UUIDv5)

### 3.1. Immutable RWDKN Namespace
```python
import json
import uuid

# Verified UUIDv5 of uuid.NAMESPACE_DNS and "realkg.nlm.nih.gov"
RWDKN_NAMESPACE_UUID = uuid.UUID("f472c728-48ba-5c19-b566-cc6217064e33")
```

### 3.2. Choice Group ID Algorithm
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

### 3.3. Regimen Option ID Algorithm
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

### 3.4. Fixed Test Vectors
```python
# Test Vector 1: Semaglutide Adjunct Regimen Group
assert make_choice_group_id(
    "urn:uuid:6ba7b810-9dad-11d1-80b4-00c04fd430c8",
    "all",
    "indicated-only-with"
) == "urn:uuid:d12c8b82-9388-51f6-9f89-8d769851ce30"

# Test Vector 2: Semaglutide + (Diet + Exercise) Option
assert make_regimen_option_id(
    "urn:uuid:d12c8b82-9388-51f6-9f89-8d769851ce30",
    "combination",
    "RXCUI:1991302",
    ["SNOMEDCT:284071006", "SNOMEDCT:229065009"]
) == "urn:uuid:e36cb07f-fb3e-5300-98b7-6f81df2220f8"
```

---

## 4. Physical Wire Contract: Manifest-Bound Sidecar (`regimens.jsonl`)

The export layout preserves existing 20-column `nodes.tsv` and 23-column `edges.tsv` while providing rich nested regimen structures in `regimens.jsonl`:

```text
application/kg-data/kgx/dailymed/
├── nodes.tsv                 # Standard 20-column KGX Nodes (includes TherapyContext nodes)
├── edges.tsv                 # Standard 23-column KGX Edges (TREATS edge with edge.id)
├── regimens.jsonl            # Structured RegimenRecord JSON lines (Accepted & Quarantined)
└── export_manifest.json      # Checksums, sizes, and record counts for all 3 artifacts
```

### 4.1. `RegimenRecord` Discriminated Union Schema

#### Branch A: Accepted Regimen Record (`record_status: "accepted"`)
```json
{
  "schema_version": "1.0.0",
  "association_id": "urn:uuid:6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "therapy_relationship": "indicated-only-with",
  "clinical_disposition": "permitted",
  "record_status": "accepted",
  "evidence_sentence_id": "s-ozempic-ind-01",
  "raw_evidence_text": "as an adjunct to diet and exercise to improve glycemic control in adults with type 2 diabetes mellitus",
  "regimen_group": {
    "id": "urn:uuid:d12c8b82-9388-51f6-9f89-8d769851ce30",
    "selection_behavior": "all",
    "options": [
      {
        "id": "urn:uuid:e36cb07f-fb3e-5300-98b7-6f81df2220f8",
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

#### Branch B: Quarantined Regimen Record (`record_status: "quarantined"`)
```json
{
  "schema_version": "1.0.0",
  "association_id": "urn:uuid:8fa6c721-1eb4-4c8d-90c1-34e89f01a2b3",
  "therapy_relationship": "indicated-only-with",
  "clinical_disposition": "permitted",
  "record_status": "quarantined",
  "quarantine_reason": "unmapped_concomitant_phrase",
  "evidence_sentence_id": "s-invokana-ind-02",
  "raw_evidence_text": "as an adjunct to standard diabetic care including behavioral coaching",
  "unresolved_terms": ["behavioral coaching"],
  "proposed_regimen_group": {
    "selection_behavior": "all",
    "options": [
      {
        "regimen_type": "combination",
        "primary_therapy": "RXCUI:1373458",
        "unresolved_therapies": ["behavioral coaching"]
      }
    ]
  },
  "regimen_group": null
}
```

---

## 5. Multi-Phase Implementation Plan

### Phase 0: Schema, LinkML, Context & Generated Contracts
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/biolink-bridge/`)

1. **Update LinkML Model (`subprojects/biolink-bridge/schema/rwdkn-model.yaml`)**:
   * Version bump to `model_version: "0.8.0"`.
   * Add `RegimenRecord`, `RegimenChoiceGroup`, `RegimenOption`, and `ProhibitedRegimen` classes.
   * Add `ActionSelectionBehaviorEnum`, `TherapyRelationshipEnum`, `RegimenTypeEnum`, `RecordStatusEnum`, and `ClinicalDispositionEnum`.
   * Add prefixes `fhir-select` and `fhir-rel`.
2. **Master Context & JSON-LD Expansion (`resources/master_context.jsonld`)**:
   * Map `fhir-select`, `fhir-rel`, `rwdkn:RegimenChoiceGroup`, `rwdkn:RegimenOption`, `rwdkn:selectionBehavior`, `rwdkn:therapyRelationship`, `rwdkn:primaryTherapy`, and `rwdkn:concomitantTherapy`.
3. **Update Contract Generator (`gen_contracts.py`)**:
   * Generate `regimen_record.schema.json` with discriminated `oneOf` branches for `accepted` and `quarantined` records.
   * Include `regimen_record.schema.json` in `resources/generated/MANIFEST.json`.
4. **Register SHACL Overlay (`subprojects/biolink-bridge/resources/rwdkn-regimen-shapes.ttl`)**:
   * Bind `rwdkn-regimen-shapes.ttl` in generated contracts and verify with `test_shacl_validation.py`.

---

### Phase 1: Stage 2 Extraction, Concept Normalization & Context Node Lifecycle
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/assertion-engine/`, `subprojects/concept-align/`, `subprojects/biolink-bridge/`)

1. **Stage 2 Assertion Extraction (`subprojects/assertion-engine/`)**:
   * Extract structured regimen modifier objects with exact sentence span bindings and sentence IDs.
2. **Concept Normalization (`subprojects/concept-align/`)**:
   * Ground non-drug therapies using the Pinned SNOMED CT Decision Table (Section 2.2).
   * Emit normalized assertions with grounded CURIEs; route ungrounded terms to quarantined payload with preserved candidate structures.
3. **KGX Context Node Serialization (`subprojects/biolink-bridge/export_kgx.py`)**:
   * Emit `TherapyContext` nodes (`SNOMEDCT:284071006`, `SNOMEDCT:229065009`, `SNOMEDCT:429300007`) in `nodes.tsv` with category `biolink:Procedure` and `node_type: "TherapyContext"`.

---

### Phase 2: Pipeline Export & Quality Guardian Validation
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/biolink-bridge/`, `subprojects/quality-guardian/`)

1. **Export Engine (`export_kgx.py`)**:
   * Emit `regimens.jsonl` companion sidecar.
   * Record SHA-256, byte size, and row/record count for `nodes.tsv`, `edges.tsv`, and `regimens.jsonl` in `export_manifest.json`.
2. **Quality Guardian Validation Gate (`validate_stage4_ontology.py`)**:
   * Validate `regimens.jsonl` against `regimen_record.schema.json`.
   * Enforce count invariant: `total_candidates == accepted + quarantined + rejected`.

---

### Phase 3: Dual Serving Projections (Neo4j LPG + Apache Jena Fuseki RDF)
**Target Repositories:** `rwdkn-data-pipeline`, `rwdkn-service`

1. **Consolidated Neo4j Loader (`load_kgx_to_neo4j.py` & `rwdkn-service/.../neo4j_loader.py`)**:
   * Materialize reified graph topology for accepted records:
     ```cypher
     (:Drug {id: 'RXCUI:1991302'})-[:TREATS {id: 'urn:uuid:assoc-101'}]->(:Disease {id: 'SNOMEDCT:44054006'})
     (:Association {id: 'urn:uuid:assoc-101'})-[:HAS_REGIMEN_CHOICE]->(:RegimenChoiceGroup {
         id: 'urn:uuid:d12c8b82-9388-51f6-9f89-8d769851ce30',
         selection_behavior: 'all',
         therapy_relationship: 'indicated-only-with'
     })
     (:RegimenChoiceGroup {id: 'urn:uuid:d12c8b82-9388-51f6-9f89-8d769851ce30'})-[:HAS_OPTION]->(:RegimenOption {
         id: 'urn:uuid:e36cb07f-fb3e-5300-98b7-6f81df2220f8',
         regimen_type: 'combination',
         primary_therapy: 'RXCUI:1991302'
     })
     (:RegimenOption {id: 'urn:uuid:e36cb07f-fb3e-5300-98b7-6f81df2220f8'})-[:PRIMARY_THERAPY]->(:Drug {id: 'RXCUI:1991302'})
     (:RegimenOption {id: 'urn:uuid:e36cb07f-fb3e-5300-98b7-6f81df2220f8'})-[:CONCOMITANT_THERAPY]->(:KGXNode:Procedure:TherapyContext:Therapy {
         id: 'SNOMEDCT:284071006',
         name: 'Dietary treatment for disorder'
     })
     (:RegimenOption {id: 'urn:uuid:e36cb07f-fb3e-5300-98b7-6f81df2220f8'})-[:CONCOMITANT_THERAPY]->(:KGXNode:Procedure:TherapyContext:Therapy {
         id: 'SNOMEDCT:229065009',
         name: 'Exercise therapy'
     })
     ```
   * Add secondary label `:Therapy` to all therapy context nodes.

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
         rwdkn:permittedRegimenGroup <urn:uuid:d12c8b82-9388-51f6-9f89-8d769851ce30> .

     <urn:uuid:d12c8b82-9388-51f6-9f89-8d769851ce30> a rwdkn:RegimenChoiceGroup ;
         rwdkn:selectionBehavior fhir-select:all ;
         rwdkn:therapyRelationship fhir-rel:indicated-only-with ;
         rwdkn:regimenOption <urn:uuid:e36cb07f-fb3e-5300-98b7-6f81df2220f8> .

     <urn:uuid:e36cb07f-fb3e-5300-98b7-6f81df2220f8> a rwdkn:RegimenOption ;
         rwdkn:regimenType rwdkn:CombinationRegimen ;
         rwdkn:primaryTherapy <https://mor.nlm.nih.gov/RxNav/search?searchBy=RXCUI&searchTerm=1991302> ;
         rwdkn:concomitantTherapy <http://snomed.info/id/284071006> ,
                                  <http://snomed.info/id/229065009> .
     ```

3. **Staged Release Publication & Service Delivery (`rwdkn-service`)**:
   * Two-phase release publication:
     1. Staging and verifying release manifest digests in Neo4j and Fuseki dataset `realkg`.
     2. Verifying option-level parity before activating live release.
   * Expose internal Flask endpoint `/api/entities/{curie}/regimens` returning structured regimen choice options.

---

### Phase 4: Automated Option-Level Parity Gate (`biolink-bridge/tests/`)
* **[NEW] `test_regimen_codec.py`**: Fixed test vector validation, deterministic UUIDv5 properties, and 8-scenario gold set.
* **[NEW] `test_regimen_choice_shacl.py`**: Automated SHACL validation via `pyshacl`.
* **[NEW] `test_cq5_dual_store_parity.py`**: Live option-level parity test asserting mathematical set equality between Neo4j and Fuseki.

---

## 6. Multi-Scenario Clinical Gold Test Matrix

Acceptance testing includes an explicit 8-scenario versioned gold test suite (`tests/fixtures/regimen_gold_set.json`):

| Scenario ID | Test Case Category | Clinical Description / Input Text | Action Selection | Therapy Relationship | Clinical Disposition | Expected Concomitant CURIEs | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GS-01` | Positive Adjunct Pair | *"as an adjunct to diet and exercise to improve glycemic control in T2D"* | `all` | `indicated-only-with` | `permitted` | `["SNOMEDCT:229065009", "SNOMEDCT:284071006"]` | `accepted` |
| `GS-02` | Explicit Monotherapy | *"indicated as monotherapy for the treatment of hypertension"* | `exactly-one` | `indicated-only-with` | `permitted` | `[]` | `accepted` |
| `GS-03` | Unspecified (Absence) | *"indicated for the treatment of major depressive disorder"* | `N/A` (No record emitted) | `N/A` | `N/A` | `[]` | `N/A` |
| `GS-04` | Single Required Therapy | *"indicated in combination with metformin for glycemic control"* | `all` | `indicated-only-with` | `permitted` | `["RXCUI:6809"]` (Metformin) | `accepted` |
| `GS-05` | Disjunctive Choice `(A+B) OR (C+D)` | *"indicated with diet and exercise, or with metformin and diet"* | `exactly-one` | `indicated-only-with` | `permitted` | 2 Options: `(Diet+Exercise)`, `(Metformin+Diet)` | `accepted` |
| `GS-06` | Contraindicated Combination | *"contraindicated in combination with GLP-1 receptor agonists"* | `all` | `contraindicated-only-with` | `contraindicated` | `["MED-RT:N0000175574"]` (GLP-1 RA EPC Class) | `accepted` |
| `GS-07` | Unresolved Phrase | *"as an adjunct to behavioral coaching"* | `N/A` | `indicated-only-with` | `permitted` | `[]` (`unresolved_terms: ["behavioral coaching"]`) | `quarantined` |
| `GS-08` | Legacy Epoch-3/0.7.0 Export | Pre-regimen DailyMed export without `regimens.jsonl` | `N/A` | `N/A` | `N/A` | `[]` | `unspecified` |

---

## 7. Verification Commands & Acceptance Criteria

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
* [ ] Pinned SNOMED CT March 2026 labels (`Dietary treatment for disorder`, `Exercise therapy`, `Lifestyle modification`) verified.
* [ ] UUIDv5 identity matches fixed test vectors with zero collisions.
* [ ] Missing regimen data remains `unspecified` and is never converted to monotherapy.
* [ ] `regimens.jsonl` checksum, size, and counts recorded in `export_manifest.json`.
* [ ] Quarantined records preserve raw evidence text without dropping base treatment edges.
* [ ] Option-level Cypher and SPARQL parity test passes with 100% tuple equality.
* [ ] ISSUE-9 is closed only upon full passing of the acceptance suite.
