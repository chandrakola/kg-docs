# Implementation Plan: Standards-Aligned Regimen Choice Architecture for CQ-5 (ISSUE-9)

**Document ID:** `RWDKN-PLAN-2026-08-30-CQ5-REV2`  
**Date:** August 30, 2026  
**Status:** PROPOSED (Incorporates all F-01 through F-17 Review Findings)  
**Supersedes:** `RWDKN-PLAN-2026-08-30-CQ5-REV1`  
**Related Documents:**  
* Reviewer Feedback: [`2026-08-30-cq5-regimen-choice-group-feedback.md`](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/2026-08-30-cq5-regimen-choice-group-feedback.md)  
* Technical Review: [`2026-08-30-cq5-regimen-choice-implementation-plan-review.md`](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/2026-08-30-cq5-regimen-choice-implementation-plan-review.md)  
* Target Repositories: `rwdkn-data-pipeline`, `rwdkn-service`, `kg-docs`  

---

## 1. Executive Summary & Core Semantic Invariants

### 1.1. Purpose
This document provides the definitive, normative implementation plan to resolve **ISSUE-9 / CQ-5 (Conditional & Adjunct Indications)**. It models complex clinical conditionality (e.g. *"indicated as an adjunct to diet and exercise for glycemic control in Type 2 Diabetes Mellitus"*) using a standards-aligned, compositional **`RegimenChoiceGroup` + `RegimenOption`** architecture.

### 1.2. Non-Negotiable Semantic Invariants
1. **Open-World Semantics (Absence $\ne$ Monotherapy)**:
   * A treatment assertion lacking a regimen group means **regimen unspecified**, *never* monotherapy.
   * Monotherapy must be explicit, positive, and backed by source evidence (`ExplicitMonotherapy`).
2. **Indivisible Option Bundles**:
   * Permitted therapy combinations (e.g. `(Diet + Exercise)` or `(Drug X + Metformin)`) are encapsulated as distinct `RegimenOption` units. They are never flattened into a single list to prevent invalid cross-combination synthesis.
3. **Exact Canonical FHIR Vocabulary**:
   * Reuse exact case-sensitive, hyphenated codes from HL7 FHIR R5:
     * Selection behavior: `http://hl7.org/fhir/action-selection-behavior` (`exactly-one`, `all`, `all-or-none`, `at-most-one`, `one-or-more`, `any`)
     * Therapy relationship: `http://hl7.org/fhir/therapy-relationship-type` (`indicated-only-with`, `contraindicated-only-with`, `replace-other-therapy`, etc.)
4. **Deterministic Identity (UUIDv5)**:
   * `choice_group_id = UUIDv5(NAMESPACE_RWDKN, association_id + ":" + selection_behavior)`
   * `regimen_option_id = UUIDv5(NAMESPACE_RWDKN, choice_group_id + ":" + sorted_therapies_string)`
5. **Non-Lossy Terminology Quarantine**:
   * If a concomitant therapy term cannot be grounded to a standard CURIE, the **base treatment association is accepted**, while the regimen modifier is quarantined for review, preventing clinical evidence loss.

---

## 2. Terminology & Standards Binding Table

All concepts must use verified ontologies. Unverified free text is prohibited in normalized output:

| Clinical Concept / Role | Canonical CURIE | Preferred Label | Ontology / Code System | Semantic Justification |
| :--- | :--- | :--- | :--- | :--- |
| **Dietary Modification** | `SNOMEDCT:284071006` | *Dietary education* | SNOMED CT (US Edition) | Prescribes structured dietary intervention in clinical labels |
| **Diet Therapy / Regimen** | `SNOMEDCT:386261001` | *Diet therapy* | SNOMED CT (US Edition) | Broader nutritional regimen category |
| **Exercise Therapy** | `SNOMEDCT:229065009` | *Exercise therapy* | SNOMED CT (US Edition) | Physical exercise intervention |
| **Lifestyle Modification** | `SNOMEDCT:429300007` | *Lifestyle modification* | SNOMED CT (US Edition) | Combined behavioral/lifestyle counseling |
| **Active Drug Substance** | `RXCUI:<id>` / `UNII:<code>` | *Ingredient Name* | RxNorm / FDA UNII | Pinned IN/PIN ingredient-level drug concept |
| **Selection Behavior** | `fhir-select:<code-id>` | *FHIR Action Selection* | HL7 FHIR R5 | Combinatorial choice logic (`exactly-one`, `all`, etc.) |
| **Therapy Relationship** | `fhir-rel:<code-id>` | *FHIR Therapy Relationship* | HL7 FHIR R5 | Indication prerequisite (`indicated-only-with`, etc.) |

---

## 3. Physical Wire Serialization Contract (Manifest-Bound Sidecar)

To prevent breaking the closed 23-column KGX TSV contract, the physical export format uses a **manifest-bound structured companion artifact (`regimens.jsonl`)**:

```text
application/kg-data/kgx/dailymed/
├── nodes.tsv                 # Standard 23-column KGX Nodes
├── edges.tsv                 # Standard 23-column KGX Edges (TREATS edge with r.id)
├── regimens.jsonl            # Structured RegimenChoiceGroup & RegimenOption records
└── export_manifest.json      # Checksums for nodes, edges, and regimens.jsonl
```

### 3.1. `regimens.jsonl` Record Schema
Each line in `regimens.jsonl` is a JSON record keyed to the canonical edge `association_id`:

```json
{
  "schema_version": "1.0.0",
  "association_id": "urn:uuid:6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "therapy_relationship": "indicated-only-with",
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
            "name": "Dietary education",
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
  },
  "status": "accepted",
  "evidence_sentence_id": "s-ozempic-ind-01"
}
```

---

## 4. Multi-Phase Implementation Plan

### Phase 0: Schema & LinkML Contract Modeling
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/biolink-bridge/`)

1. **Update LinkML Model (`subprojects/biolink-bridge/schema/rwdkn-model.yaml`)**:
   * Add `RegimenChoiceGroup` and `RegimenOption` classes with explicit FHIR code mappings.
   * Add `ActionSelectionBehaviorEnum` with canonical values (`all`, `any`, `all-or-none`, `exactly-one`, `at-most-one`, `one-or-more`).
   * Add `TherapyRelationshipEnum` with canonical values (`indicated-only-with`, `contraindicated-only-with`, `replace-other-therapy`, etc.).
   * Add `RegimenTypeEnum` with compositional values (`combination`, `monotherapy`).
2. **Generate Structural Contracts**:
   * Run `gen_contracts.py` to regenerate JSON Schemas, context, and validation IR.
3. **Add SHACL Overlay (`subprojects/biolink-bridge/resources/rwdkn-regimen-shapes.ttl`)**:
   * Enforce graph shape integrity: options must have primary therapy; combination options must have $\ge 1$ concomitant therapy; monotherapy must have 0 concomitant therapies.

---

### Phase 1: Stage 2 Extraction & Concept Normalization
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/assertion-engine/`, `subprojects/concept-align/`)

1. **Stage 2 LLM Prompt Updates (`assertion-engine`)**:
   * Update prompt schema to output structured `regimen` objects alongside SPO triples.
   * Inherit `primary_therapy` deterministically from triple subject (`RXCUI`).
2. **Concept Normalization (`concept-align`)**:
   * Normalize non-drug therapy terms using the Terminology Binding Table (Section 2).
   * Implement non-lossy quarantine: if a concomitant therapy cannot be grounded, tag the regimen record as `status: "quarantined"` with `raw_text` and route for manual review, while safely emitting the base treatment edge.

---

### Phase 2: Pipeline Export & Quality Guardian Gates
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/biolink-bridge/`, `subprojects/quality-guardian/`)

1. **Export Sidecar (`export_kgx.py`)**:
   * Emit `regimens.jsonl` alongside `nodes.tsv` and `edges.tsv`.
   * Record `regimens_sha256`, `accepted_regimens_count`, and `quarantined_regimens_count` in `export_manifest.json`.
2. **Quality Guardian Validation Gate (`validate_stage4_ontology.py`)**:
   * Verify schema validity of `regimens.jsonl`.
   * Verify that all therapy CURIEs match approved vocabularies and that deterministic UUIDv5 rules are respected.

---

### Phase 3: Dual Serving Projections (Neo4j LPG + Apache Jena Fuseki RDF)
**Target Repositories:** `rwdkn-data-pipeline`, `rwdkn-service`

1. **Consolidated Neo4j Loader (`load_kgx_to_neo4j.py` & `rwdkn-service/.../neo4j_loader.py`)**:
   * Materialize reified regimen topology in Neo4j:
     ```cypher
     (:Drug {id: 'RXCUI:1991302'})-[:TREATS {id: 'urn:uuid:assoc-101'}]->(:Disease {id: 'SNOMEDCT:44054006'})
     (:Association {id: 'urn:uuid:assoc-101'})-[:HAS_REGIMEN_CHOICE]->(:RegimenChoiceGroup {id: 'urn:uuid:rcg-01', selection_behavior: 'all'})
     (:RegimenChoiceGroup {id: 'urn:uuid:rcg-01'})-[:HAS_OPTION]->(:RegimenOption {id: 'urn:uuid:ro-01', regimen_type: 'combination'})
     (:RegimenOption {id: 'urn:uuid:ro-01'})-[:CONCOMITANT_THERAPY]->(:Therapy {id: 'SNOMEDCT:284071006', name: 'Dietary education'})
     (:RegimenOption {id: 'urn:uuid:ro-01'})-[:CONCOMITANT_THERAPY]->(:Therapy {id: 'SNOMEDCT:229065009', name: 'Exercise therapy'})
     ```
   * Add uniqueness constraints and indexes on `(Association.id)`, `(RegimenChoiceGroup.id)`, and `(RegimenOption.id)`.
2. **RDF / Fuseki Projection (`rdf_projection.py` & `rwdkn-service`)**:
   * Project reified Turtle / N-Triples:
     ```turtle
     @prefix biolink:     <https://w3id.org/biolink/vocab/> .
     @prefix fhir-select: <http://hl7.org/fhir/action-selection-behavior/> .
     @prefix rwdkn:       <https://w3id.org/realkg/> .

     <urn:uuid:assoc-101> a biolink:ChemicalToDiseaseOrPhenotypicFeatureAssociation ;
         biolink:subject <https://mor.nlm.nih.gov/RxNav/search?searchBy=RXCUI&searchTerm=1991302> ;
         biolink:predicate biolink:treats ;
         biolink:object <http://snomed.info/id/44054006> ;
         rwdkn:permittedRegimenGroup <urn:uuid:rcg-01> .

     <urn:uuid:rcg-01> a rwdkn:RegimenChoiceGroup ;
         rwdkn:selectionBehavior fhir-select:all ;
         rwdkn:regimenOption <urn:uuid:ro-01> .

     <urn:uuid:ro-01> a rwdkn:RegimenOption ;
         rwdkn:regimenType rwdkn:CombinationRegimen ;
         rwdkn:primaryTherapy <https://mor.nlm.nih.gov/RxNav/search?searchBy=RXCUI&searchTerm=1991302> ;
         rwdkn:concomitantTherapy <http://snomed.info/id/284071006> ,
                                  <http://snomed.info/id/229065009> .
     ```
   * Include Fuseki dataset publication and verification in `rwdkn-service/scripts/install_release.py`.

---

### Phase 4: Automated CQ-5 Dual-Graph Parity Gate
**Target Repository:** `rwdkn-data-pipeline` (`subprojects/biolink-bridge/tests/`)

Implement `test_cq5_dual_store_parity.py` comparing canonical option-level tuples:

#### 4.1. Option-Level Canonical Parity Tuple
```python
@dataclass(frozen=True)
class RegimenOptionTuple:
    association_id: str
    drug_curie: str
    indication_curie: str
    choice_group_id: str
    selection_behavior: str
    regimen_option_id: str
    regimen_type: str
    primary_therapy_curie: str
    concomitant_therapies: tuple[str, ...]  # Alphabetically sorted CURIEs
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
     opt.id AS regimen_option_id,
     opt.regimen_type AS regimen_type,
     d.id AS primary_therapy_curie,
     t.id AS therapy_curie
ORDER BY therapy_curie
RETURN association_id, drug_curie, indication_curie, choice_group_id, selection_behavior,
       regimen_option_id, regimen_type, primary_therapy_curie,
       collect(DISTINCT therapy_curie) AS concomitant_therapies;
```

#### 4.3. Exact SPARQL Parity Query
```sparql
PREFIX biolink:     <https://w3id.org/biolink/vocab/>
PREFIX fhir-select: <http://hl7.org/fhir/action-selection-behavior/>
PREFIX rwdkn:       <https://w3id.org/realkg/>

SELECT ?association_id ?drug_iri ?indication_iri ?choice_group_id ?selection_behavior
       ?regimen_option_id ?regimen_type ?primary_therapy_iri
       (GROUP_CONCAT(DISTINCT STR(?therapy_iri); separator="|") AS ?concomitant_therapies_concat)
WHERE {
    ?assoc a biolink:ChemicalToDiseaseOrPhenotypicFeatureAssociation ;
           biolink:subject ?drug_iri ;
           biolink:predicate biolink:treats ;
           biolink:object ?indication_iri ;
           rwdkn:permittedRegimenGroup ?rcg .
    
    ?rcg rwdkn:selectionBehavior ?selection_behavior ;
         rwdkn:regimenOption ?opt .
    
    ?opt rwdkn:regimenType ?regimen_type ;
         rwdkn:primaryTherapy ?primary_therapy_iri .
    
    OPTIONAL {
        ?opt rwdkn:concomitantTherapy ?therapy_iri .
    }
    BIND(STR(?assoc) AS ?association_id)
    BIND(STR(?rcg) AS ?choice_group_id)
    BIND(STR(?opt) AS ?regimen_option_id)
}
GROUP BY ?association_id ?drug_iri ?indication_iri ?choice_group_id ?selection_behavior
         ?regimen_option_id ?regimen_type ?primary_therapy_iri
```

The test maps SPARQL IRIs to CURIEs, sorts the concomitant therapy lists, and asserts exact mathematical set equality against Neo4j results.

---

## 5. Verification Commands & Acceptance Criteria

```bash
# 1. Verify LinkML schema and regenerate contract artifacts
cd subprojects/biolink-bridge && uv run python gen_contracts.py && uv run python gen_contracts.py --check

# 2. Run SHACL structural validation on regimen graphs
uv run pytest tests/test_regimen_choice_shacl.py -v

# 3. Run unit tests on sidecar serialization and deterministic UUIDv5 rules
uv run pytest tests/test_regimen_serialization.py -v

# 4. Run live dual-store CQ-5 parity integration gate
uv run pytest tests/test_cq5_dual_store_parity.py -v

# 5. Run Quality Guardian gate checks
cd ../quality-guardian && uv run pytest tests/
```

### Acceptance Checklist
* [ ] All FHIR selection and therapy relationship codes use exact case-sensitive, hyphenated standards.
* [ ] Concomitant therapies use verified SNOMED-CT / RxNorm CURIEs from the Terminology Table.
* [ ] Missing regimen data remains `unspecified` and is never converted to monotherapy.
* [ ] `regimens.jsonl` is checksum-bound in `export_manifest.json`.
* [ ] Unresolved therapy phrases are quarantined with evidence without dropping base SPO edges.
* [ ] Option-level Cypher and SPARQL parity test passes with 100% tuple equality on Semaglutide, Tirzepatide, and Liraglutide.
* [ ] ISSUE-9 is closed only upon full passing of the acceptance suite.
