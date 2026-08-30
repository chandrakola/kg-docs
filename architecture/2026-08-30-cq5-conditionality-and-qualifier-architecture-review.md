# Technical Review — CQ-5 Conditionality and Qualifier Architecture

**Prepared:** 2026-08-30  
**Reviewer role:** Independent architecture and implementation reviewer  
**Reviewed documents:**

- `kg-docs/architecture/2026-08-30-cq5-conditionality-and-qualifier-architecture.md`
- `rwdkn-data-pipeline/docs/2026-08-30-cq5-conditionality-and-qualifier-architecture.md`

**Copy check:** The two reviewed files are byte-for-byte identical (SHA-256 `a41806fd28addaecd41814a95bfd3df839b2a7dfce9819be8bf00f01f239407b`).  
**Verdict:** **Approve the architectural direction, but revise the contract and examples before implementation. ISSUE-9 remains open.**

## Executive assessment

The proposal identifies a real and important defect: treatment assertions preserve conditional wording in evidence, but CQ-5 cannot yet distinguish adjunct therapy from monotherapy using structured graph properties. The proposed end-to-end ownership—from extraction through normalization, KGX, Neo4j, RDF, and query acceptance—is the right shape.

The document is not implementation evidence. Commit `966b44e` adds only this proposal, its status is `PROPOSED`, and the four proposed fields are absent from implementation files. The live PostgreSQL database contains nine evidence records with “adjunct to diet and exercise,” while live Neo4j has 344 `TREATS` relationships and zero `condition_qualifier` or `population_qualifier` properties. This is a useful baseline: it proves the source signal exists and the structured graph gap remains.

Before Epoch-4 work starts, the document should resolve four design questions: the physical wire representation, reuse versus replacement of existing qualifier names, local RWDKN extensions versus Biolink slots, and the semantics of unknown versus explicitly false/monotherapy values.

## What is strong

1. **The problem statement is evidence-backed.** The Semaglutide KGX edge has the correct `biolink:treats` assertion and verbatim adjunct wording, but its current qualifications are only `confidence`, `assertion_status`, and `clinical_modality`.
2. **The proposal preserves assertion identity.** Conditionality belongs on the reified association/relationship, not on the drug or disease node.
3. **The proposed stage ownership is broadly correct.** Extraction should identify the fact, normalization should canonicalize it, quality gates should reject or quarantine invalid values, and both serving projections should expose the same semantics.
4. **It keeps evidence alongside discrete fields.** Structured qualifiers should supplement—not replace—the source sentence and sentence/triple identifiers.
5. **It supplies CQ-shaped target queries.** This is the correct way to define parity once the examples are aligned with the actual graph contract.

## Findings requiring revision

### F-01 — The document closes ISSUE-9 before any implementation exists

The title says “Closing ISSUE-9,” and the status section says the issue is closed and CQ-5 resolved, while the same document says `PROPOSED` and presents all four phases as future work. The adding commit contains no schema, extractor, exporter, loader, projector, validator, or test changes.

**Recommendation:** Use `Status: PROPOSED / IMPLEMENTATION PENDING`. Change the issue statement to “design addresses ISSUE-9; close only after the dual-store acceptance gate passes.”

### F-02 — The Epoch-4 wire contract is ambiguous

“23+ Column KGX TSV” does not say whether the four fields become new physical columns or canonical keys inside the existing `qualifications` column. That decision changes the schema version, migration path, generated contracts, readers, manifests, and fixtures.

The current pipeline already serializes allowlisted structured qualifiers into `qualifications`; the Neo4j loader parses those keys and materializes them as relationship properties. RDF currently emits only the entire `qualifications` string as one `rwdkn:qualifications` literal.

**Recommendation:** Choose and document one contract explicitly. The lower-risk path is:

- keep the 23-column physical wire;
- add canonical fields to the semantic triple model and controlled vocabulary;
- encode their normalized values as named `qualifications` keys;
- let Neo4j continue materializing parsed keys; and
- update RDF projection to parse the same keys into discrete predicates.

If dedicated TSV columns are required, declare an exact Epoch-4 header and an Epoch-3-to-4 migration/downgrade policy instead of “23+.”

### F-03 — `population_qualifier` duplicates the existing population contract

The assertion engine already normalizes canonical `population` values, and the bridge renames them to `population_context_qualifier`. Adding `population_qualifier` creates two competing names and vocabularies. The proposed `adults_and_pediatrics_ge_10` value also combines multiple populations and an age threshold in one token, while the slot is declared single-valued.

**Recommendation:** Reuse and extend one canonical population model. At minimum, specify migration aliases and one authoritative vocabulary. Prefer separable fields such as population category plus age boundary/operator rather than generating a new enum value for every age/population combination.

### F-04 — The proposal overstates Biolink 4.3.6 alignment

`condition_qualifier`, `population_qualifier`, `combination_therapy_qualifier`, and `line_of_therapy_qualifier` are not the names of four standardized slots in the pinned Biolink 4.3.6 snapshot. The RDF example correctly places the first two in the `rwdkn:` namespace, but the diagram calls `biolink:condition_qualifier`, and the prose calls all four standardized Biolink slots.

Biolink 4.3.6 does define `population context qualifier`, but its range is an ontology class—not a literal such as `"adults"`. It also marks `severity qualifier` deprecated and gives `onset qualifier` a narrow phenotypic-use scope, so the existing qualifier mapping deserves care rather than expansion by analogy.

**Recommendation:** State that these are RWDKN extension slots informed by Biolink qualifier patterns. Give each slot a `slot_uri`, range, extension rationale, and mapping status. Either use local literal-valued RWDKN predicates consistently or use valid CURIE-valued Biolink slots; do not mix the two representations under one name.

### F-05 — The conditionality model conflates independent clinical dimensions

The proposed `ConditionQualifierEnum` mixes regimen role (`monotherapy`), required non-drug context (`adjunct_to_diet_and_exercise`), background therapy (`adjunct_to_standard_of_care`), and prevention intent (`secondary_prevention`). These dimensions can coexist and will grow combinatorially. A boolean `combination_therapy_qualifier` also cannot identify the co-therapy and cannot distinguish explicitly false from unknown/unstated.

**Recommendation:** Separate at least:

- `regimen_role`: `monotherapy`, `adjunct`, `combination`, or `unknown`;
- `required_context`: multivalued controlled concepts such as diet and exercise;
- `co_therapy`: zero or more normalized therapy/drug CURIEs or reviewed text values;
- `line_of_therapy`: a controlled enum, not an unconstrained string; and
- population/age constraints using the existing population contract.

Do not infer `monotherapy` or `combination_therapy=false` merely because adjunct wording is absent. Missing, explicitly false, and explicitly stated monotherapy must remain distinct.

### F-06 — Field names are inconsistent inside the proposal

The schema defines `combination_therapy_qualifier`, but the extraction JSON and evidence table use `combination_therapy`. The flow diagram uses `biolink:condition_qualifier`, while the RDF example and SPARQL query use `rwdkn:condition_qualifier`.

**Recommendation:** Add a canonical field dictionary showing the Stage-1 JSON key, normalized key, KGX encoding, Neo4j property, RDF predicate, datatype/range, cardinality, null semantics, writer, validator, and migration alias for every field.

### F-07 — The query examples do not match the current graph

The current serving data uses:

- `MED-RT:N0000178480` / `GLP-1 Receptor Agonist`, not the Cypher name `GLP-1 Receptor Agonists [EPC]`;
- RDF IRI `.../MEDRT/N0000178480`, not `N0000175570`;
- `source_section=INDICATIONS_AND_USAGE`, while `34084-4` is carried in sentence/edge identity rather than `source_section`; and
- generic `biolink:Association` typing in the current RDF projector, not `biolink:ChemicalToDiseaseOrPhenotypicFeatureAssociation`.

Consequently, the displayed Cypher and SPARQL are target sketches, not currently executable acceptance queries. Even after adding condition fields, the class identifier/type mismatches would still return no results.

**Recommendation:** Use stable class IDs rather than names, correct the MED-RT code, preserve both normalized section name and LOINC section code as distinct fields if both are needed, and either update RDF association-category typing or query the currently guaranteed `biolink:Association` contract.

### F-08 — The evidence and acceptance claims need measurable scope

“Evidence Preservation (100% Captured)” is too broad unless it is scoped to a defined CQ-5 cohort and backed by counts. The roadmap’s “test against 12 labels” and “100% gate pass” measure pipeline acceptance, not extraction quality. A model can emit valid enum values and still classify clinical meaning incorrectly.

**Recommendation:** Build a versioned, sentence-level gold set containing positive, negative, ambiguous, explicit-monotherapy, adjunct-to-drug, adjunct-to-lifestyle, combination-product, age-qualified, and multi-indication examples. Report precision, recall, exact-match performance, unknown/review rate, and evidence-span grounding. Include at least one adversarial example where “monotherapy” occurs in a safety table rather than an indication.

## Required acceptance gates

1. The proposal identifies one canonical name and representation for every qualifier across all stages.
2. Missing, false, unknown, and explicit monotherapy semantics are tested independently.
3. A migration test proves the selected Epoch-3-to-4 wire behavior and protects existing 23-column consumers.
4. Assertion-engine tests cover normalization, synonyms, conjunctions, conflicting clauses, and `qualifier_review` fallback.
5. Concept-align tests prove qualifiers survive identity and concept normalization unchanged unless explicitly normalized.
6. Quality Guardian rejects invalid controlled values and reports them without silently dropping source evidence.
7. Neo4j and RDF projections derive discrete properties/predicates from the same canonical payload.
8. Live Cypher and SPARQL CQ-5 queries run against the same release and return equivalent canonical tuples.
9. Query tests use the real class CURIE, association type, source-section contract, and release-scoped data.
10. ISSUE-9 closes only after the gold-set quality threshold and live dual-store parity gate pass.

## Verification snapshot

| Check | Observed result |
|---|---|
| Document copies | Identical |
| Adding commit `966b44e` | Documentation only |
| Current proposal fields in implementation code | None found |
| PostgreSQL evidence containing adjunct phrase | 9 rows |
| Live Neo4j `TREATS` relationships | 344 |
| Live Neo4j relationships with `condition_qualifier` | 0 |
| Live Neo4j relationships with `population_qualifier` | 0 |
| Biolink bridge tests | 289 passed |
| Assertion clinical-qualifier tests | 9 passed |

## Final recommendation

**Proceed with Epoch-4 design work after revising this specification; do not mark CQ-5 or ISSUE-9 resolved yet.**

The proposal has the correct goal and pipeline boundary. The most important revision is to freeze a single canonical qualifier contract that reuses existing population handling, distinguishes regimen role from required context, states null/unknown semantics, and produces executable parity queries against the identifiers and association types the system actually serves.
