---
type: architectural-review-feedback
title: "Third-Party AI Architectural Review Prompt Feedback — Portability Round 1"
description: "Evidence-first feedback on the NLM-KN/RWDKN multi-cloud portability review prompt."
slug: third-party-ai-architectural-review-feedback-portability
round: 1
algorithm: evidence-first-repository-grounded-architecture-review
created: 2026-08-10
updated: 2026-08-10
timestamp: "2026-08-10 10:59:11 EDT (-0400)"
status: review-feedback
source_prompt: "/home/chandrakola/.gemini/antigravity-ide/brain/a79ae057-a8c5-44f6-9201-81884f8a6a87/third_party_ai_architectural_review_prompt.md"
workspace: "/home/chandrakola/development/NLM/kg"
tags: [third-party-review, architecture, portability, multi-cloud, rwdkn, nlm-kn, evidence-first]
links:
  - "../../rwdkn-data-pipeline/docs/specs/architecture.md"
  - "../../rwdkn-service/docs/specs/architecture.md"
  - "../../rwdkn-service/docs/real-kg-demo-operations.md"
  - "../../POST-REBOOT.md"
---

# Third-Party AI Architectural Review Prompt Feedback

**Prepared:** 2026-08-10 10:59:11 EDT  
**Review slug:** `third-party-ai-architectural-review-feedback-portability`  
**Round:** 1  
**Algorithm:** `evidence-first-repository-grounded-architecture-review`  
**Reviewer stance:** independent third-party reviewer; no implementation changes made

## Executive assessment

The prompt is a useful review scaffold, but it currently asks an AI reviewer to validate a
desired architecture rather than independently establish the architecture that exists.

The live checkout documents `real-kg` as a primarily file-based, replayable pipeline using
shared home-lab services. It also contains separate `rwdkn-data-pipeline` and `rwdkn-service`
repositories, deployment orchestration, and operational artifacts. Several claims in the prompt
are stronger than the available evidence, especially:

- universal MinIO/S3 mirroring;
- isolated production Postgres and Neo4j as the normal serving boundary;
- atomic schema swap with zero downtime;
- complete code/infrastructure decoupling from POSIX paths and host assumptions.

The prompt should force the reviewing AI to distinguish verified facts from documented intent,
assumptions, and contradictions before making recommendations.

## What is strong in the prompt

- It identifies the important architectural themes: portability, staging/serving isolation,
  data promotion, and distributed quota coordination.
- It requests actionable short-term and long-term recommendations.
- It recognizes that deployment topology and data movement are coupled concerns.
- It gives reviewers enough conceptual context to discuss cloud, database, and operational tradeoffs.

## Findings requiring revision

### F-01 — The prompt treats the target architecture as established fact

The diagram and key decisions are written declaratively. This encourages the external model to
review the diagram instead of checking whether the repository, compose files, manifests, scripts,
and runtime actually implement it.

**Recommendation:** add an evidence classification requirement:

```markdown
Do not assume that the architecture description is accurate. First construct an
evidence table from repository files, manifests, compose files, deployment scripts,
runtime configuration, and operational documentation. Mark each major claim as
VERIFIED, DOCUMENTED-ONLY, ASSUMED, or CONTRADICTED. Cite the file and line or
runtime observation supporting the classification.
```

### F-02 — “S3 protocol abstraction” is not sufficient evidence of portability

The prompt claims that every run is mirrored to MinIO and that standard S3 URIs provide
100% cloud portability. That conclusion is too strong. S3-compatible systems differ in IAM,
TLS, endpoint discovery, region handling, path-style addressing, multipart upload behavior,
checksums, object metadata, lifecycle rules, consistency expectations, and presigned URL behavior.

The review must also inspect non-S3 dependencies, including:

- NFS and `$SHARE_WRITE` paths;
- Docker bind mounts, external volumes, and external networks;
- `localhost`, `127.0.0.1`, and container-name defaults;
- host-mounted timezone and filesystem paths;
- Infisical bootstrap credentials and endpoint reachability;
- UMLS/RxNav/SNOMED data availability, licensing, and versioning;
- Gemini-specific APIs, model slugs, batch semantics, and quota behavior.

**Recommendation:** replace “100% cloud portability” with a measurable portability contract:
supported targets, allowed adapters, unchanged application layers, target-specific configuration,
and acceptance tests for each deployment target.

### F-03 — The dual-plane description may not match current ownership boundaries

The prompt describes a shared staging data plane and an isolated application data plane. Current
repository material describes a file-based ETL pipeline using shared services and a separate
application/federation orchestration path. `manage-apps.sh` delegates federation startup; that
does not by itself prove that all serving databases are isolated from staging databases.

**Recommendation:** require the reviewer to produce an ownership matrix for:

| Resource | Creating owner | Writing owner | Reading owner | Network boundary | Durable location |
|---|---|---|---|---|---|
| Pipeline artifacts |  |  |  |  |  |
| Postgres |  |  |  |  |  |
| Neo4j |  |  |  |  |  |
| Mongo limiter state |  |  |  |  |  |
| Object storage |  |  |  |  |  |
| Secrets |  |  |  |  |  |

### F-04 — The promotion claim needs to be tested, not accepted

The prompt assumes a `shared-postgres → rwdkn-postgres` dump-and-schema-swap flow with zero
downtime. A reviewer should verify the actual commands and confirm whether the process includes:

- tables, views, indexes, sequences, extensions, permissions, and constraints;
- a coherent dataset version shared by Postgres and Neo4j;
- transfer duration and temporary disk requirements at 100M+ edges;
- reader cutover and connection draining;
- rollback after a partial restore or failed graph load;
- RPO, RTO, checksums, and artifact retention;
- whether “atomic” means transactional, filesystem-level, routing-level, or merely procedural.

The operational documentation should be treated as evidence of the implemented process, not as
proof that the stronger zero-downtime property holds.

### F-05 — The concurrency question presents a false three-way choice

MongoDB leases, Redis locks, and an API gateway are not interchangeable:

- a lease store coordinates ownership and expiry;
- a rate limiter controls admission or token consumption;
- a gateway limits ingress but may not coordinate asynchronous provider jobs;
- a scheduler can provide durable retries, fairness, and quota accounting.

The reviewer should first identify the required semantics: global quota scope, lease duration,
crash recovery, fencing tokens, idempotency, provider response handling, auditability, and whether
the limiter must survive process and host failure. Only then should MongoDB, Redis, a gateway, or
a dedicated scheduler be compared.

### F-06 — Code/infrastructure decoupling is overstated

Environment variables and secret injection improve separation, but they do not eliminate
infrastructure coupling. The review should explicitly search for filesystem paths, Docker network
names, volume assumptions, service names, local ports, host-mounted files, provider-specific
health checks, and startup ordering.

It should also assess whether Infisical itself is a runtime dependency and how the system behaves
during bootstrap, secret rotation, Infisical outage, or migration to cloud-native identity systems.

### F-07 — Important data and knowledge-graph concerns are absent

The review is infrastructure-heavy and underweights KG correctness and reproducibility. Add review
questions for:

- KGX/Biolink and schema-version compatibility;
- ontology database version and licensing reproducibility;
- provenance from source artifact to exported node and edge;
- deterministic run manifests and checksums;
- model/provider/version capture for LLM extraction;
- replaying a run without repeating paid LLM calls;
- synchronized promotion of graph data and relational evidence data;
- validation and rollback of rejected or partially accepted triples.

### F-08 — Reliability, security, and operations are under-specified

Add explicit review of:

- backup and restore testing;
- disaster recovery and cross-region recovery;
- RPO/RTO targets;
- resource sizing and noisy-neighbor protection;
- observability, alerting, and audit logs;
- secret rotation and least privilege;
- network exposure and tenant isolation;
- upgrade compatibility for Postgres, Neo4j, MongoDB, and object storage;
- operational ownership when a shared service fails.

## Recommended prompt structure

Add this section before the four core evaluation areas:

```markdown
## EVIDENCE-FIRST REVIEW METHOD

1. Inventory the supplied repositories, deployment files, manifests, environment
   contracts, operational docs, and runtime evidence.
2. Build a current-state ownership and dependency matrix.
3. Classify each system-overview claim as VERIFIED, DOCUMENTED-ONLY, ASSUMED,
   or CONTRADICTED.
4. Separate current state, intended state, and proposed state.
5. For every portability claim, name the unchanged component, environment-specific
   adapter, and acceptance test.
6. Do not describe a promotion as atomic or zero-downtime unless the cutover,
   rollback, and failure behavior are evidenced.
7. State confidence and missing evidence for each conclusion.
```

Then retain the existing four areas, but add the KG correctness, provenance, DR, security, and
operational ownership areas above.

## Suggested title change

Current title:

> NLM-KN / RWDKN Multi-Cloud Environment Isolation

Suggested title:

> Evidence-Based Portability and Deployment-Boundary Review for NLM-KN/RWDKN

This avoids making “multi-cloud isolation” sound like an already-verified implementation state.

## Review conclusion

**Disposition: Revise before sharing with external AI reviewers.**

The prompt’s topics are appropriate, but its strongest claims need to become hypotheses. The most
valuable improvement is the evidence-first method: require every AI reviewer to identify what is
implemented, what is merely documented, and what remains a portability target. Without that guard,
the resulting reviews are likely to be polished but architecturally detached from the live code.
