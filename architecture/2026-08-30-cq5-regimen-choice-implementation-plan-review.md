# Technical Review — CQ-5 Regimen Choice Implementation Plan

**Prepared:** 2026-08-30  
**Reviewer role:** Independent architecture and implementation-plan reviewer  
**Reviewed document:** `2026-08-30-cq5-regimen-choice-implementation-plan.md`  
**Reviewed document ID:** `RWDKN-PLAN-2026-08-30-CQ5-REV1`  
**Verdict:** **Revise before implementation. The core model is approved, but the current plan is not executable as written. ISSUE-9 remains open.**

## Executive assessment

The plan makes the correct major architectural move: it replaces an unbounded list of composite condition strings with `RegimenChoiceGroup` and indivisible `RegimenOption` resources. It also correctly preserves the open-world rule that missing regimen data is unspecified rather than monotherapy.

The implementation plan nevertheless has several blocking gaps. Some values identified as FHIR codes are not present in the cited FHIR code system; the two proposed KGX wire options are unresolved and neither is compatible with the current pipeline without additional design; the Cypher and SPARQL parity queries are not equivalent; one SNOMED CT example is incorrect; and the plan does not cover the actual `rwdkn-service` release loader or a concrete Fuseki load/query path.

The recommended disposition is:

> Accept the `RegimenChoiceGroup` architecture, revise the implementation contract, and begin coding only after the wire representation, standard-code mappings, stable identifiers, failure policy, and serving ownership are frozen.

## What is strong

1. **The open-world invariant is correct.** The plan explicitly states that absence of a group is unspecified and never inferred monotherapy.
2. **Regimen bundles preserve combinatorial integrity.** Keeping `(A + B)` and `(C + D)` as separate options prevents accidental creation of `A + D` or `B + C`.
3. **The association is correctly reified for RDF.** Regimen semantics qualify the treatment assertion rather than the drug or disease independently.
4. **The plan recognizes both semantic and physical validation.** LinkML, generated contracts, SHACL, and Quality Guardian are appropriate layers when their ownership is clarified.
5. **The intended dual-store parity gate is valuable.** Neo4j and RDF must derive from one canonical regimen payload and return equivalent option-level results.
6. **The phase ordering is broadly reasonable.** Schema, extraction, normalization, serialization, projection, and acceptance testing are the right workstreams.

## Blocking findings

### F-01 — The proposed FHIR therapy-relationship enum is not standards-conformant

The plan lists:

```text
indicated_only_with
contraindicated_with
replaces
adjunct_to
```

FHIR R5 does not define `contraindicated-with`, `replaces`, or `adjunct-to`. Its complete therapy-relationship code system includes values such as:

```text
contraindicated-only-with
contraindicated-except-with
indicated-only-with
indicated-except-with
indicated-only-after
indicated-only-before
replace-other-therapy
replace-other-therapy-contraindicated
replace-other-therapy-not-tolerated
replace-other-therapy-not-effective
```

The official system is `http://hl7.org/fhir/therapy-relationship-type`, and it is case-sensitive and complete. See [FHIR Therapy Relationship Type](https://www.hl7.org/fhir/R5/codesystem-therapy-relationship-type.html).

**Required revision:** Import or reproduce the exact FHIR codes with their canonical hyphenated lexical forms. Do not label local aliases as FHIR codes. If internal snake-case names are retained, define an explicit, tested one-to-one mapping to the FHIR system and code.

### F-02 — FHIR action-selection codes are changed into local spellings without a mapping contract

FHIR defines `all-or-none`, `exactly-one`, `at-most-one`, and `one-or-more`, not `all_or_none`, `exactly_one`, `at_most_one`, and `one_or_more`. The official code system is also case-sensitive and complete. See [FHIR Action Selection Behavior](https://fhir.hl7.org/fhir/R5/codesystem-action-selection-behavior.html).

**Required revision:** Store the canonical FHIR code and code-system URI. A LinkML permissible value may use a safe internal key only if its `meaning` or equivalent mapping points to the exact FHIR concept and every RDF/API projection emits the canonical hyphenated code.

### F-03 — The wire contract is left as an implementation-time choice

Phase 3 offers Option A or Option B without selecting one. This is not a minor implementation detail; it determines schema compatibility, identifiers, graph topology, migration behavior, and every downstream consumer.

The current physical contract is a closed 23-column `AssociationRecord`. Its `qualifications` value is a pipe-delimited `key=value` string. The Neo4j loader parses those entries into scalar relationship properties, while the RDF projector currently emits the complete `qualifications` field as an opaque literal. Nested JSON is therefore not currently round-trip safe or queryable without a new parser and versioned encoding.

Option B is also incompatible with the current KGX contract as written. `AssociationRecord.predicate` permits only `biolink:*` predicates, generated validation checks predicates against pinned Biolink 4.3.6, and regimen classes and connecting predicates are local RWDKN constructs rather than current Biolink entities and predicates.

**Required revision:** Freeze one wire contract before implementation. The lowest-risk option is a manifest-bound structured companion artifact, for example:

```text
nodes.tsv
edges.tsv
regimens.jsonl
export_manifest.json
```

Each regimen record should be keyed by the canonical association ID, validated by a generated schema, checksum-bound in the export manifest, and consumed by both the RDF and Neo4j projectors. This preserves KGX compatibility without burying nested data in an opaque string or pretending local regimen topology is standard KGX.

If the project instead selects embedded JSON or explicit KGX nodes, the plan must specify the exact escaping, schema version, compatibility behavior, parsing failure policy, and conformance implications.

### F-04 — Stable identifier generation is unspecified

Examples use identifiers such as `urn:uuid:cg-01`, `urn:uuid:rcg-01`, and `urn:uuid:ro-01`. These are illustrative labels, not valid UUID URNs, and no deterministic generation rule is provided.

Random IDs would make reruns non-idempotent and make RDF/Neo4j parity difficult to prove.

**Required revision:** Define deterministic identifiers, for example UUIDv5 values derived from:

```text
choice-group ID = UUIDv5(association ID + canonical group semantics)
regimen-option ID = UUIDv5(choice-group ID + canonical ordered option payload)
```

Canonicalization must define ordering, duplicate handling, terminology versions, and whether evidence identity participates in the hash.

### F-05 — One terminology mapping is demonstrably incorrect

The plan maps diet to `SNOMEDCT:226060000` and labels it “Dietary education.” That code represents **Stress management**, not diet or dietary education. The plan also provides alternatives with “or” but does not define a preferred concept, semantic equivalence, terminology edition, or deterministic selection policy.

The exercise example `SNOMEDCT:229065009` does represent Exercise therapy. The diet concept still requires terminology review against the intended meaning: diet regimen, nutrition therapy, diet education, and dietary modification are not interchangeable.

**Required revision:** Remove unverified identifiers from the normative plan. Add a terminology decision table containing source phrase, selected CURIE, preferred label, terminology edition/version, semantic rationale, alternatives rejected, and reviewer. Assertions with unresolved therapy concepts must be preserved and routed for review rather than silently discarded.

### F-06 — The extraction example does not satisfy the proposed schema

The proposed `RegimenOption` requires `primary_therapy` and `selection_behavior`, but the extraction JSON emits neither within the option. The plan does not identify whether the assertion engine, concept-align, or the bridge derives the primary therapy from the triple subject.

It also calls assertion extraction “Stage 1,” while the current pipeline contract treats Stage 1 as acquisition and Stage 2 as assertion extraction.

**Required revision:** Publish one canonical JSON example that validates against the proposed LinkML schema and identify the writer of every field. Use the pipeline’s existing stage names. If `primary_therapy` is deterministically inherited from the association subject, state that rule and test it instead of requiring the LLM to repeat the value.

### F-07 — “Reject ungrounded free text” creates an evidence-loss failure mode

Fail-closed terminology validation is appropriate before publishing structured therapy identifiers, but rejecting the entire treatment assertion would discard a valid base assertion and its evidence simply because a contextual therapy could not be normalized.

**Required revision:** Separate these outcomes:

```text
base association accepted
regimen structure accepted
regimen structure quarantined for terminology review
entire assertion rejected for an independent contract violation
```

Preserve the raw therapy phrase, source span, candidate mappings, decision status, and review reason. Do not expose unresolved text as a normalized therapy IRI, but do not lose the source assertion.

### F-08 — The Neo4j topology is not supported by the current loaders

The pipeline loader currently creates KGX entity nodes and direct predicate relationships. It does not create `Association`, `RegimenChoiceGroup`, or `RegimenOption` nodes. The plan therefore requires more than adding relationship properties.

There is also a second active loader in `rwdkn-service/services/shared/neo4j_loader.py`, invoked by `rwdkn-service/scripts/install_release.py`. The plan targets only `rwdkn-data-pipeline/subprojects/biolink-bridge/load_kgx_to_neo4j.py`, creating a likely release-install/runtime divergence.

**Required revision:** Identify one owning loader or update both with shared fixtures and parity tests. Add uniqueness constraints/indexes for association, choice-group, and option IDs; idempotent merge behavior; source-scoped cleanup; rollback behavior; and migration tests.

### F-09 — The proposed Neo4j shortcut property is lossy

`r.regimen_type = 'combination'` cannot represent an association whose choice group permits both explicit monotherapy and a combination option. It also creates a second source of truth that may drift from the reified regimen topology.

**Required revision:** Do not persist a single `regimen_type` shortcut on the `TREATS` relationship. If a performance hint is proven necessary, use a non-lossy derived summary such as `has_regimen_constraints=true` or a multivalued set, document it as derived, and test regeneration from canonical regimen options.

### F-10 — The RDF type and native-FHIR boundary remain ambiguous

The current projector emits `biolink:Association`; the plan changes the type to `biolink:ChemicalToDiseaseOrPhenotypicFeatureAssociation` without specifying category-selection logic or updating the semantic contract. It also uses FHIR code IRIs as a lightweight projection rather than native FHIR RDF coding nodes.

Both choices can be valid, but they must be explicit.

**Required revision:** Either retain the guaranteed `biolink:Association` type or define and test deterministic association-category specialization. State that the shown RDF is an RWDKN semantic projection aligned to FHIR code systems, and document a separate mapping if native FHIR RDF exchange is required.

### F-11 — The Cypher and SPARQL parity queries are incorrect and collapse regimen boundaries

The SPARQL query selects `?drug`, `?indication`, and `?therapy`, but the graph pattern binds `?d`, `?dis`, and `?t`. The selected variables are therefore unbound. It also groups only by drug, indication, and behavior, causing therapies from multiple regimen options to be concatenated into one flattened result—the exact error the architecture is intended to prevent.

The Cypher query similarly aggregates all therapies across every matched option because it does not return or group by the option ID. It compares labels rather than stable identifiers, while SPARQL returns IRIs. `GROUP_CONCAT` ordering is not a reliable basis for tuple equality.

**Required revision:** Define parity at option granularity using canonical identifiers:

```text
association_id
choice_group_id
group_selection_behavior
regimen_option_id
regimen_type
primary_therapy_curie
sorted concomitant_therapy_curie set
```

Normalize Neo4j CURIEs and RDF IRIs through the existing prefix map before comparison. Compare sets as sets, not concatenated labels.

### F-12 — Fuseki is named as a live gate without an installation or loading path

The reviewed repositories contain the RDF projector but no identified Fuseki configuration, dataset loader, release installer, readiness probe, or service query adapter. The plan names `rwdkn-service` as a target repository but assigns no implementation phase to it.

**Required revision:** Add explicit service work for:

- release artifact installation;
- RDF dataset replacement or versioned graph loading;
- transactional/atomic publication behavior;
- endpoint configuration and authentication;
- readiness checks;
- query timeout and failure handling;
- release identity checks proving Neo4j and Fuseki serve the same export; and
- the service/API query path that exposes CQ-5 results.

If Fuseki is not currently an operational serving component, classify the parity test as a staged integration target rather than a current live gate.

## Additional findings

### F-13 — Prohibition is not a regimen composition type

`prohibited_regimen` appears inside `RegimenTypeEnum` beside `combination_regimen` and `explicit_monotherapy`. “Prohibited” describes permission or safety disposition, while “combination” and “monotherapy” describe composition. These dimensions should not share one enum.

**Recommendation:** Keep `RegimenOption` composition separate from a disposition such as permitted, prohibited, contraindicated, not recommended, or unsupported. Do not equate all prohibited combinations with FHIR `contraindicated-only-with` unless the source is actually a contraindication.

### F-14 — Generated-contract wording and commands are inaccurate

`gen_contracts.py` currently produces seven total artifacts, not seven JSON schemas. Two are JSON Schema documents; the others include OWL, SHACL, JSON-LD context, validation IR, and the manifest.

The verification command `gen_contracts.py --check` checks for drift but does not perform the regeneration described by its comment. The listed SHACL command runs the existing `test_shacl_validation.py`, not the newly proposed `test_regimen_choice_shacl.py`.

**Recommendation:** Separate regeneration and zero-diff checking, and list every new test explicitly in the verification commands.

### F-15 — Hand-authored and generated SHACL ownership can drift

The plan adds `rwdkn-regimen-shapes.ttl`, while the repository already has generated base SHACL and an existing hand-authored SHACL resource. It does not say whether the new file is generated, imported, checksum-bound, or executed by Quality Guardian.

**Recommendation:** Generate constraints from LinkML where expressible. Put only cross-property rules in a clearly named SHACL overlay, include that overlay in the contract manifest, and prove that both offline validation and the production gate load the same base-plus-overlay shape set.

### F-16 — The test cohort is almost entirely positive and does not test the architecture’s difficult cases

Semaglutide, tirzepatide, and liraglutide are useful adjunct examples, but they do not prove monotherapy, mixed options, prohibited combinations, ambiguity, or absence semantics. The listed liraglutide identifier `RXCUI:897122` is a clinical drug product (“3 ML liraglutide 6 MG/ML Pen Injector”), whereas the ingredient identifier used elsewhere in the project is `RXCUI:475968`.

**Recommendation:** Select test identifiers at one declared RxNorm term-type level and add a versioned gold set covering:

- explicit monotherapy;
- one required concomitant therapy;
- diet and exercise as one indivisible option;
- `(A + B) OR (C + D)`;
- monotherapy or a specific combination;
- exactly-one, at-most-one, and one-or-more selection;
- explicit contraindicated/prohibited combination;
- “not recommended” without contraindication;
- ambiguous or unresolved therapy terminology;
- no regimen statement;
- conflicting statements across sections; and
- legacy records created before the regimen contract.

Report extraction precision, recall, exact option-bundle match, terminology accuracy, unknown/review rate, and evidence-span grounding—not only schema-valid output counts.

### F-17 — No migration and backfill policy is defined

The architecture changes existing DailyMed assertions and graph topology, but the plan does not say whether historical artifacts will be re-extracted, deterministically backfilled from evidence, or left unspecified.

**Recommendation:** Define an epoch/version transition, backfill scope, old-consumer behavior, source-scoped reload procedure, rollback artifact, and release manifest marker. Legacy absence must remain unspecified and must never be converted to monotherapy.

## Recommended revised implementation path

### Phase 0 — Freeze decisions and fixtures

1. Correct all FHIR and terminology values.
2. Select the physical wire representation.
3. Define deterministic IDs and canonical ordering.
4. Define permitted, prohibited, unknown, and unresolved semantics.
5. Approve a small gold set before prompt or schema changes.

### Phase 1 — Canonical semantic contract

1. Add LinkML regimen classes and slots.
2. Preserve canonical FHIR system-plus-code pairs.
3. Separate regimen composition from permission/safety disposition.
4. Generate structural contracts.
5. Add a checksum-bound SHACL overlay only for constraints LinkML cannot express.

### Phase 2 — Extraction and normalization

1. Extend the existing Stage-2 assertion contract.
2. Preserve raw therapy phrases and evidence spans.
3. Normalize therapies in concept-align with terminology version metadata.
4. Quarantine unresolved regimen structures without dropping valid base associations.
5. Validate exact option bundles against the gold set.

### Phase 3 — Backward-compatible export

1. Keep `nodes.tsv` and `edges.tsv` compatible.
2. Emit a versioned, schema-validated `regimens.jsonl` keyed by association ID.
3. Add file checksum, record count, accepted/quarantined counts, and schema identity to `export_manifest.json`.
4. Prove deterministic regeneration and round-trip fidelity.

### Phase 4 — Shared serving projections

1. Make Neo4j and RDF consume the same regimen artifact.
2. Materialize deterministic association/group/option identity.
3. Update both the pipeline loader and the service release installer, or consolidate ownership.
4. Add Fuseki publication and release-identity checks.
5. Update the service query adapter/API contract for option-level results.

### Phase 5 — Acceptance

1. Run unit, contract, SHACL, migration, and idempotency tests.
2. Run gold-set extraction and terminology evaluation.
3. Compare canonical option-level tuples between Neo4j and RDF.
4. Verify unknown versus explicit monotherapy behavior.
5. Verify one released snapshot end to end through the service API.
6. Close ISSUE-9 only after all gates pass.

## Required acceptance gates

1. Every claimed FHIR value is an exact code from the declared FHIR version or is explicitly marked as an RWDKN extension.
2. The physical serialization is singular, versioned, deterministic, and manifest-bound.
3. Repeated runs generate identical association, choice-group, and option IDs.
4. Missing regimen information remains unspecified at every stage and in both stores.
5. Explicit monotherapy is accepted only with source evidence.
6. Option boundaries survive extraction, normalization, export, RDF projection, Neo4j loading, and querying.
7. Ungrounded therapies are quarantined with evidence rather than silently published or discarded.
8. Both Neo4j loaders—or one consolidated owner—produce identical topology.
9. Neo4j and RDF parity compares stable option-level identifiers and sorted therapy sets.
10. The RDF and Neo4j stores prove they serve the same release manifest.
11. The service API exposes the result without requiring callers to issue raw Cypher or SPARQL.
12. Migration and rollback are tested on pre-regimen artifacts.
13. Gold-set quality thresholds pass for positive, negative, mixed, ambiguous, and adversarial cases.
14. ISSUE-9 remains open until the end-to-end acceptance run succeeds.

## Final recommendation

**Do not implement REV1 unchanged.** The `RegimenChoiceGroup` architecture should proceed, but first publish REV2 with the blocking decisions resolved.

The highest-priority REV2 changes are:

1. Correct the FHIR and SNOMED terminology.
2. Choose one backward-compatible wire contract; a manifest-bound regimen sidecar is the cleanest fit with the current KGX profile.
3. Define deterministic identifiers and a non-lossy unresolved-term policy.
4. Correct parity queries to preserve individual regimen options.
5. Add the real service loader, Fuseki publication, query adapter, migration, and gold-set work.

Once those items are explicit, the plan will be implementable without reintroducing the original hardcoded-enum problem or creating inconsistent RDF and LPG representations.
