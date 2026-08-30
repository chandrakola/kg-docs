# Reviewer Feedback — Standards-Aligned Regimen Choice Architecture for CQ-5

**Prepared:** 2026-08-30  
**Reviewer role:** Independent architecture reviewer  
**Related issue:** ISSUE-9 / CQ-5 — Discrete Graph Conditionality and Context Qualifiers  
**Related proposal:** `2026-08-30-cq5-conditionality-and-qualifier-architecture.md`  
**Recommendation:** Replace the open-ended condition-qualifier enumeration with a standards-aligned regimen choice structure before implementation.

## Executive judgment

The proposal should not maintain composite values such as:

```text
adjunct_to_diet_and_exercise
adjunct_to_standard_of_care
adjunct_to_radiotherapy
```

That approach will expand whenever a new therapy, conjunction, or permitted combination appears. It also combines several independent meanings—selection logic, regimen role, and therapy identity—inside one string.

The preferable model is a reified `RegimenChoiceGroup` containing one or more explicit `RegimenOption` resources. Each option identifies the therapies that belong together, while the group defines how options may be selected. This preserves particular combinations without creating a configuration file or enumeration value for every regimen.

Use existing standards wherever they fit:

- Biolink for the treatment association and biomedical identifiers.
- FHIR `ClinicalUseDefinition` for indication and relationship-to-other-therapy semantics.
- FHIR `PlanDefinition.action.selectionBehavior` values for regimen selection logic.
- SNOMED CT, RxNorm, or another governed terminology for therapies.
- PROV-O and existing evidence identifiers for provenance.
- SHACL for RDF structural validation.

Only the small RDF structure connecting these standards needs to remain in the `rwdkn:` namespace.

## Important correction: absence does not mean monotherapy

The rule below is unsafe and should not be adopted:

```text
No RegimenChoiceGroup = monotherapy
RegimenChoiceGroup exists = combination therapy
```

RDF uses open-world semantics. A missing `RegimenChoiceGroup` can mean:

- the source did not state the regimen;
- extraction did not identify it;
- the record predates this architecture;
- the source evidence is ambiguous; or
- the projection lost the structured qualifier.

Therefore:

```text
No RegimenChoiceGroup = regimen unspecified
```

Monotherapy must be represented only when supported explicitly by source evidence. Missing, unknown, and explicit monotherapy are different states.

## Recommended conceptual model

```text
TreatmentAssociation
├── subject: Drug
├── predicate: treats
├── object: Disease
├── evidence and provenance
└── permittedRegimenGroup: RegimenChoiceGroup
    ├── selectionBehavior: exactly-one
    ├── option: RegimenOption 1
    │   ├── regimenType: combination
    │   ├── primaryTherapy: Drug
    │   ├── concomitantTherapy: Therapy A
    │   └── concomitantTherapy: Therapy B
    └── option: RegimenOption 2
        ├── regimenType: combination
        ├── primaryTherapy: Drug
        ├── concomitantTherapy: Therapy C
        └── concomitantTherapy: Therapy D
```

The choice group and its options are first-class resources because the qualifier applies to the complete treatment assertion, not independently to the drug or disease.

## Selection behavior

Do not create local `rwdkn:allOf` and `rwdkn:anyOf` codes when FHIR already provides the required selection vocabulary:

```text
all
any
all-or-none
exactly-one
at-most-one
one-or-more
```

The relevant standard is [FHIR Action Selection Behavior](https://hl7.org/fhir/R5/valueset-action-selection-behavior.html).

FHIR `all` means that all actions in an option are selected as one unit. FHIR `exactly-one` means that one and only one option must be selected. Nested groups can therefore express complex permitted combinations without inventing composite qualifier values.

## RDF example: exactly one permitted combination

The following example means that the permitted choices are `(A + B)` or `(C + D)`, and arbitrary cross-combinations are not allowed:

```turtle
@prefix biolink: <https://w3id.org/biolink/vocab/> .
@prefix fhir-rel: <http://hl7.org/fhir/therapy-relationship-type/> .
@prefix fhir-select: <http://hl7.org/fhir/action-selection-behavior/> .
@prefix prov: <http://www.w3.org/ns/prov#> .
@prefix rwdkn: <https://example.nlm.nih.gov/rwdkn/vocab/> .
@prefix ex: <https://example.nlm.nih.gov/rwdkn/resource/> .

ex:treatment-association-123
    a biolink:Association ;
    biolink:subject ex:drug-X ;
    biolink:predicate biolink:treats ;
    biolink:object ex:disease-Y ;
    rwdkn:therapyRelationship fhir-rel:indicated-only-with ;
    rwdkn:permittedRegimenGroup ex:choice-group-123 ;
    rwdkn:sourceText
        "Use only as one of the specified combination regimens." ;
    prov:wasDerivedFrom ex:evidence-123 .

ex:choice-group-123
    a rwdkn:RegimenChoiceGroup ;
    rwdkn:selectionBehavior fhir-select:exactly-one ;
    rwdkn:regimenOption ex:regimen-AB,
                         ex:regimen-CD .

ex:regimen-AB
    a rwdkn:RegimenOption ;
    rwdkn:regimenType rwdkn:CombinationRegimen ;
    rwdkn:selectionBehavior fhir-select:all ;
    rwdkn:primaryTherapy ex:drug-X ;
    rwdkn:concomitantTherapy ex:therapy-A,
                              ex:therapy-B .

ex:regimen-CD
    a rwdkn:RegimenOption ;
    rwdkn:regimenType rwdkn:CombinationRegimen ;
    rwdkn:selectionBehavior fhir-select:all ;
    rwdkn:primaryTherapy ex:drug-X ;
    rwdkn:concomitantTherapy ex:therapy-C,
                              ex:therapy-D .
```

The example uses a lightweight FHIR-aligned RDF projection, not the complete native FHIR RDF serialization. The canonical application model can map the same structure to FHIR `ClinicalUseDefinition` and `PlanDefinition` when full FHIR exchange is required.

## Why regimen bundles must not be flattened

The example must not be flattened into one list:

```turtle
rwdkn:requiredTherapy ex:therapy-A,
                      ex:therapy-B,
                      ex:therapy-C,
                      ex:therapy-D ;
rwdkn:selectionBehavior fhir-select:any .
```

That representation can be interpreted as permitting invalid choices such as `A`, `B + C`, or `A + D`. Keeping each approved combination in its own `RegimenOption` prevents consumers from synthesizing combinations that the evidence never authorized.

## Explicit monotherapy

For a source that explicitly states monotherapy, represent the fact positively and preserve its evidence:

```turtle
ex:treatment-association-456
    a biolink:Association ;
    biolink:subject ex:drug-X ;
    biolink:predicate biolink:treats ;
    biolink:object ex:disease-Y ;
    rwdkn:regimenType rwdkn:ExplicitMonotherapy ;
    rwdkn:sourceText "Drug X is indicated as monotherapy for Disease Y." ;
    prov:wasDerivedFrom ex:evidence-456 .
```

Do not infer this value from the absence of concomitant therapies.

## Mixed monotherapy and combination choices

The structure must support indications that allow either monotherapy or a specific combination:

```text
RegimenChoiceGroup: exactly-one
├── RegimenOption 1: explicit monotherapy
│   └── primary therapy: Drug X
└── RegimenOption 2: combination
    ├── primary therapy: Drug X
    └── concomitant therapy: Drug Z
```

This is another reason why group presence cannot itself mean combination therapy. A choice group may contain monotherapy, combination therapy, or both.

## Prohibited and contraindicated combinations

Known prohibited combinations should be represented explicitly and separately from permitted regimen options:

```turtle
ex:prohibited-regimen-BD
    a rwdkn:ProhibitedRegimen ;
    rwdkn:prohibitedTherapy ex:therapy-B,
                            ex:therapy-D ;
    rwdkn:prohibitionReason
        "Concurrent use creates an unacceptable risk." ;
    prov:wasDerivedFrom ex:safety-evidence-789 .
```

Absence from the permitted list should not automatically be published as a contraindication. “Not listed,” “not recommended,” “unsupported,” and “contraindicated” have different clinical meanings and require distinct evidence.

## Minimal local vocabulary

The local vocabulary can remain small and stable:

### Classes

```text
rwdkn:RegimenChoiceGroup
rwdkn:RegimenOption
rwdkn:CombinationRegimen
rwdkn:ExplicitMonotherapy
rwdkn:ProhibitedRegimen
```

### Predicates

```text
rwdkn:permittedRegimenGroup
rwdkn:regimenOption
rwdkn:regimenType
rwdkn:selectionBehavior
rwdkn:primaryTherapy
rwdkn:concomitantTherapy
rwdkn:prohibitedTherapy
rwdkn:sourceText
```

The open-ended values—drugs, procedures, diet, exercise, radiotherapy, and other therapies—must be terminology concepts rather than additions to this local vocabulary.

## Validation requirements

Use SHACL to validate the published RDF structure. At minimum:

1. Every `RegimenChoiceGroup` has a recognized FHIR selection behavior.
2. Every group has at least one `RegimenOption`.
3. Every option identifies its primary therapy.
4. An explicit monotherapy option has supporting evidence and no asserted concomitant therapy.
5. A combination option has at least one asserted concomitant therapy.
6. Every therapy value is an IRI or a reviewed unresolved-concept resource, not an uncontrolled string.
7. Every permitted or prohibited regimen preserves source evidence and provenance.
8. Duplicate and contradictory options are rejected or routed for review.
9. Missing regimen information remains unspecified; it is never converted silently to monotherapy.

[SHACL](https://www.w3.org/TR/shacl/) validates graph structure. It is not, by itself, a patient-specific prescribing engine.

## Safety boundary

RDF should publish the clinical knowledge and its provenance. If these combinations will drive patient-specific recommendations, order entry, or alerts, the executable rules should use a governed clinical decision-support layer such as FHIR `PlanDefinition` and, where necessary, CQL or equivalent validated logic.

Dose, timing, route, age, laboratory values, renal or hepatic function, contraindications, and drug interactions must not be reduced to a simple regimen membership query when they affect safety.

## Constructive recommendation to the proposal author

Revise the CQ-5 proposal as follows:

1. Retire the open-ended `ConditionQualifierEnum` design.
2. Introduce `RegimenChoiceGroup` and `RegimenOption` as the canonical conditionality structure.
3. Reuse FHIR therapy-relationship and action-selection values rather than creating local equivalents.
4. Keep therapy identities terminology-backed and open-ended.
5. Treat absence of a regimen group as unspecified, not monotherapy.
6. Require explicit evidence for monotherapy, combination requirements, and prohibited combinations.
7. Preserve each permitted combination as an indivisible regimen option.
8. Add SHACL validation and dual-store parity tests before closing ISSUE-9.

## Final verdict

`RegimenChoiceGroup` is a materially better architecture than hardcoded regimen-role or condition-qualifier values. It is compact, compositional, queryable, and capable of preserving exact permitted combinations.

The design should be accepted with one firm semantic rule:

> A missing regimen structure means **unspecified**, never monotherapy. Monotherapy and combination constraints must both be explicit and evidence-backed.

With that correction, the approach follows current standards where standards exist and limits custom RWDKN semantics to a small structural bridge rather than an endlessly expanding clinical vocabulary.
