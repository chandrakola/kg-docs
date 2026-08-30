# Technical Review — CQ-5 Regimen Choice Implementation Plan REV3

**Prepared:** 2026-08-30  
**Reviewer role:** Independent architecture and implementation-plan reviewer  
**Reviewed file:** `docs/2026-08-30-cq5-regimen-choice-implementation-plan.md`  
**Reviewed document ID:** `RWDKN-PLAN-2026-08-30-CQ5-REV3`  
**Verdict:** **Conditionally approved architecture; correct the release-blocking contract contradictions before implementation. ISSUE-9 remains open.**

## Executive assessment

REV3 is a substantial and constructive response to the REV2 review. It now defines:

- a `RegimenRecord` sidecar contract;
- accepted and quarantined examples;
- pinned SNOMED CT terminology;
- canonical UUIDv5 recipes;
- 20-column node and 23-column edge compatibility;
- context concepts as explicit nodes;
- therapy relationship and primary therapy in both projections;
- option-level parity normalization;
- a multi-scenario gold matrix;
- migration and rollback intent; and
- a service-facing regimen endpoint.

The overall architecture is approved. No return to composite qualifiers or per-regimen configuration files is warranted.

REV3 is not yet executable unchanged because several internal contradictions would either fail validation or create release ambiguity. These are narrower than the REV2 findings and can be fixed in one final specification pass:

1. the published namespace UUID does not match its stated derivation;
2. the plan reuses model version `0.7.0` and export epoch `3`, which are already active without a regimen sidecar;
3. the gold set requires `status=prohibited`, but the declared status enum permits only `accepted` and `quarantined`;
4. context nodes are declared `node_type=TherapyContext`, while Neo4j examples and parity queries require the label `Therapy`;
5. LinkML generation of the accepted/quarantined conditional JSON branches is not specified;
6. current loaders do not provide the atomic cross-store publication claimed by the plan; and
7. the new local and FHIR prefixes/predicates are not wired into the existing generated context and strict CURIE expansion contract.

Once those items are corrected, implementation can begin without another architectural review.

## Resolution of REV2 findings

| REV2 concern | REV3 status | Reviewer judgment |
|---|---|---|
| Correct SNOMED labels | Mostly resolved | Diet and exercise labels corrected; lifestyle term still requires evidence from the pinned release |
| Collision-safe deterministic IDs | Partially resolved | Canonical recipes added, but namespace derivation and group cardinality remain problematic |
| `RegimenRecord` wrapper/schema | Resolved in direction | Wrapper and generated schema named; conditional branch mechanics remain unspecified |
| Accepted versus quarantined records | Partially resolved | Examples added; schema constraints and rejected/prohibited distinctions remain inconsistent |
| Therapy context node lifecycle | Partially resolved | Nodes now enter `nodes.tsv`; stage ownership, labels, mappings, and cleanup require alignment |
| Therapy relationship in projections | Resolved | Present in Neo4j, RDF, and parity tuple |
| Stored primary therapy | Resolved | Present in Neo4j and RDF; parity reads the stored value |
| Parity normalization | Substantially resolved | Explicit functions added; prefix/empty-value validation still needed |
| Artifact integrity | Resolved in direction | Hash, size, and counts proposed for three core artifacts |
| Generic RDF association type | Resolved | Query and projection now use `biolink:Association` |
| SHACL overlay registration | Resolved in direction | Manifest checksum proposed; production loading path still needs one owner |
| Service/Fuseki delivery | Partially resolved | Installer and endpoint named; current implementation is not atomic and route convention differs |
| Gold-set coverage | Substantially resolved | Eight scenarios added; prohibited semantics and extraction metrics need correction |
| Migration/backfill | Partially resolved | Legacy behavior stated, but current live epoch/version makes the policy incorrect |
| Evidence linkage | Improved | Raw evidence and sentence ID added; referential validation remains unstated |

## Release-blocking findings

### F-01 — The immutable namespace UUID does not match the documented derivation

REV3 states that this value is UUIDv5 of the DNS namespace and `realkg.nlm.nih.gov`:

```python
RWDKN_NAMESPACE_UUID = uuid.UUID("645391e8-7828-5690-b1fa-8a603c7e7dd6")
```

The actual calculation is:

```python
uuid.uuid5(uuid.NAMESPACE_DNS, "realkg.nlm.nih.gov")
# f472c728-48ba-5c19-b566-cc6217064e33
```

The documented value may still be adopted as an intentionally assigned namespace, but it must not be described as the result of a different calculation. Changing the namespace after data is published changes every group and option identifier.

**Required correction:** Choose one of these and freeze it:

1. use `f472c728-48ba-5c19-b566-cc6217064e33` and retain the stated derivation; or
2. retain `645391e8-7828-5690-b1fa-8a603c7e7dd6`, label it an explicitly assigned RWDKN namespace, and record its provenance in an architecture decision.

Add fixed test vectors containing payload, canonical bytes, and expected UUIDs.

### F-02 — REV3 reuses versions that already describe the current non-regimen export

The live exporter already writes:

```text
model_version: 0.7.0
kgx_export_epoch: 3
```

Those current epoch-3 releases do not contain `regimens.jsonl`. REV3 says epoch 3 requires the sidecar and treats only epoch 2 as legacy. That would make existing valid epoch-3 artifacts appear corrupt or nonconformant.

**Required correction:** Do not redefine an already published version. Use one of:

- model version `0.8.0` plus release-bundle epoch `4`; or
- preserve KGX TSV epoch `3` and add a separate required `regimen_contract_version: 1.0.0` with a new bundle/profile version.

The migration matrix must explicitly include pre-regimen epoch-3 exports and load them as regimen-unspecified.

### F-03 — Prohibited semantics cannot pass the declared sidecar schema

The LinkML plan defines:

```text
RegimenStatusEnum = accepted | quarantined
```

The gold matrix expects:

```text
GS-06 status = prohibited
```

The manifest invariant also mentions `rejected`, but neither the status enum nor manifest example defines rejected records/counts. `ProhibitedRegimen` is named as a class without a complete JSON branch.

More importantly, prohibition is clinical disposition, not pipeline processing status.

**Required correction:** Separate the axes:

```text
record_status: accepted | quarantined | rejected
clinical_disposition: permitted | contraindicated | not_recommended
regimen_type: monotherapy | combination
```

GS-06 should normally be an accepted record with `clinical_disposition=contraindicated` and the exact FHIR relationship `contraindicated-only-with`. It should not use `record_status=prohibited`.

Define whether rejected candidates are written to `regimens.jsonl` or a separate rejection artifact, then make the count invariant match that decision.

### F-04 — Therapy context labels are inconsistent across KGX and Neo4j

REV3 says context nodes are exported as:

```text
category: biolink:Procedure
node_type: TherapyContext
```

The Neo4j topology and parity query use:

```cypher
(t:Therapy)
```

The current loader derives labels from category and `node_type`; it will not automatically convert `TherapyContext` into `Therapy`. The new `(t:Therapy)` match can therefore return no rows.

**Required correction:** Freeze one graph-label contract. Prefer querying the stable category-derived label or `KGXNode` plus ID/category rather than inventing a third label. If `TherapyContext` is the canonical node type, use that consistently and add its allowed `(node_type, category)` pair to exporter mappings, LinkML, generated validation IR, tests, and both loaders.

### F-05 — The accepted/quarantined JSON Schema branches are not mechanically defined

Two examples do not establish a closed conditional schema. A single LinkML `RegimenRecord` with optional slots will ordinarily permit invalid hybrids unless the generator is customized or separate subclasses/unions are defined.

Invalid examples that must fail include:

- `status=accepted` with `regimen_group=null`;
- `status=accepted` with unresolved terms;
- `status=quarantined` without `quarantine_reason` or raw evidence;
- `status=quarantined` carrying an asserted unresolved CURIE; and
- explicit monotherapy containing a concomitant therapy.

**Required correction:** Define separate `AcceptedRegimenRecord` and `QuarantinedRegimenRecord` branches under a discriminated union, or document deterministic JSON Schema post-processing that emits `oneOf`/`if-then` constraints. Add negative schema fixtures.

### F-06 — The stated atomic publication behavior is not provided by current loaders

The current service installer loads only `nodes.tsv` and `edges.tsv`. Its Neo4j loader optionally wipes the source first and then writes in batches. That is not an atomic swap. Neo4j and Fuseki are also separate systems, so one transaction cannot cover both.

Validating all artifacts before mutation prevents one failure class, but a later connection or batch failure can still leave partial or cross-store-mismatched state.

**Required correction:** Describe this as a two-phase release publication:

1. validate and stage the complete release in inactive Neo4j/Fuseki release scope;
2. verify counts, hashes, and CQ parity;
3. activate a release pointer/alias only after both stores pass; and
4. retain the previous release for rollback.

If blue/green storage is out of scope, remove the word “atomic” and document the actual partial-failure recovery procedure.

### F-07 — Prefix and local vocabulary plumbing is missing

The plan introduces:

```text
fhir-select:
fhir-rel:
rwdkn:RegimenChoiceGroup
rwdkn:RegimenOption
rwdkn:selectionBehavior
rwdkn:therapyRelationship
rwdkn:primaryTherapy
rwdkn:concomitantTherapy
```

The current RDF projection performs strict CURIE expansion from generated prefix/context resources, and generated contracts validate emitted vocabulary. Merely adding LinkML classes is not enough.

**Required correction:** Add explicit tasks for `master_context.jsonld`, prefix expansion, local class/predicate declarations, OWL/SHACL generation, exported context copies, strict expansion tests, and RDF round-trip tests. Declare whether FHIR values are stored as system-plus-code pairs or projected to local code IRIs.

## Important constructive findings

### F-08 — Context-node stage ownership crosses the current pipeline boundary

REV3 assigns concept-align responsibility for emitting rows directly into `nodes.tsv`, but `biolink-bridge/export_kgx.py` owns KGX serialization.

**Recommendation:** Concept-align should normalize therapy concepts in the canonical assertion artifact. Biolink-bridge should create or reuse the corresponding `nodes.tsv` rows. Quality Guardian should validate the resulting endpoint/context closure.

### F-09 — Group identity still needs a cardinality invariant

The choice-group ID excludes options. It is collision-free only if an association can own at most one group for a given selection behavior and therapy relationship. Multiple source statements or merged assertions could otherwise create different group content with the same ID.

**Recommendation:** Either:

- require exactly one merged regimen record/group per association and deterministically union/deduplicate its options before ID generation; or
- include canonical option identities in the group payload.

Test duplicate source sentences, conflicting options, and changed option sets across releases.

### F-10 — The terminology pinning is incomplete

The SNOMED CT URI is concretely versioned, but “Pinned NLM RxNorm Monthly Release” does not identify a release date or version. The Biolink pin should reference the repository’s checksum-bound snapshot, not only a general model URL. The FHIR binding should state version `5.0.0` and note that the relevant code systems are Trial Use.

The `Lifestyle modification` code/label also needs an auditable lookup result from the declared March 2026 SNOMED CT release before acceptance.

**Recommendation:** Put release URI/date and checksum or artifact identity beside every terminology source and make terminology lookup part of contract generation or validation.

### F-11 — The service endpoint should follow the existing internal route convention

The live query application exposes internal routes such as:

```text
/api/entities/{curie}
/api/entities/{curie}/neighborhood
```

REV3 proposes `/explorer/api/drug/{curie}/regimens`. `/explorer` may be an external gateway prefix, but it is not the internal Flask route convention, and `drug` unnecessarily narrows a reusable entity endpoint.

**Recommendation:** Prefer internal `/api/entities/{curie}/regimens`, then document any externally mounted `/explorer` prefix separately. Add response schema, 404/empty behavior, release metadata, evidence links, authorization, and endpoint tests.

### F-12 — Gold-set scenario GS-06 uses the wrong identifier kind

“GLP-1 receptor agonists” is a pharmacologic class, not an RxNorm ingredient. `RXCUI:...` is therefore not the appropriate expected identifier placeholder for that phrase.

**Recommendation:** Use the project’s verified MED-RT/EPC class CURIE or change the fixture to name a specific concomitant drug. Ensure the option model permits therapy-context classes as well as ingredients and procedures.

### F-13 — Eight fixtures are a contract smoke set, not an extraction-quality evaluation

The matrix is a good minimum semantic suite, but zero UUID collisions across eight records is not meaningful collision evidence, and perfect store parity does not prove correct extraction.

**Recommendation:** Add:

- fixed UUID test vectors plus property tests over many generated payloads;
- negative schema fixtures;
- a clinically reviewed extraction set with precision, recall, exact-bundle match, and quarantine rate;
- adversarial wording and section-context cases; and
- terminology-version regression fixtures.

### F-14 — Quarantine loses partial structured meaning

The quarantined example sets `regimen_group` to null. For a phrase containing one mapped and one unmapped therapy, this drops the successfully resolved part from the review payload.

**Recommendation:** Preserve a non-asserted `proposed_regimen_group` or candidate structure containing resolved terms, unresolved terms, roles, confidence, and mapping decisions. Serving projections must still ignore it until accepted.

### F-15 — Evidence referential integrity remains implicit

The sidecar carries `evidence_sentence_id` and raw text, but no gate requires the sentence ID to match the referenced association’s `sentence_id` or provenance record.

**Recommendation:** Validate association existence, evidence/triple identity, source document/section, and exact source-span grounding. Define behavior for merged associations supported by multiple regimen-bearing sentences.

### F-16 — Release cleanup must include reified topology

Source-scoped reload currently focuses on ordinary nodes and direct relationships. REV3 does not define deletion or retention rules for orphaned association, choice-group, option, and context nodes.

**Recommendation:** Add ownership/source properties to reified nodes and test replacement of a release where an option is removed. Shared terminology context nodes should survive while still referenced and should not be wiped as if they were source-private assertions.

## Required final specification corrections

1. Correct or relabel the namespace UUID and publish fixed identity vectors.
2. Introduce a new bundle/model capability version without redefining current epoch 3 / model 0.7.0.
3. Separate record processing status from clinical disposition and repair GS-06/count invariants.
4. Align `TherapyContext` node type, Neo4j labels, mappings, and queries.
5. Define mechanically closed accepted/quarantined JSON Schema branches.
6. Replace the unsupported atomic claim with a concrete staged publication protocol.
7. Add prefix/context/local-vocabulary wiring tasks.
8. Restore stage ownership: concept-align normalizes; biolink-bridge serializes KGX.
9. Define one-group-per-association or include options in group identity.
10. Pin RxNorm/FHIR/Biolink artifacts precisely and verify the lifestyle code.
11. Align the service endpoint with existing `/api/entities/...` conventions.
12. Correct the prohibited-class fixture and add property/negative/clinical tests.

## Implementation authorization recommendation

The work can be split safely:

### Safe to begin after the document corrections

- LinkML semantic classes and sidecar wrapper;
- generated nested JSON Schema work;
- codec and UUID fixed-vector tests;
- accepted/quarantined negative fixtures;
- terminology lookup fixtures; and
- exporter-side sidecar generation behind a disabled feature flag.

### Do not activate until end-to-end gates pass

- mandatory sidecar enforcement;
- source wipe/reload changes;
- production Neo4j reified topology;
- Fuseki publication;
- public API exposure; and
- ISSUE-9 closure.

## Final recommendation

**Approve the `RegimenChoiceGroup` architecture and sidecar implementation direction. Require one final contract correction pass before coding against REV3 as a normative specification.**

The remaining issues are finite and concrete. They do not justify new clinical enums, additional per-regimen configuration, or another architecture pivot. Correcting identity provenance, release versioning, disposition/status semantics, node-label consistency, conditional schemas, and publication mechanics will make this plan implementation-ready.
