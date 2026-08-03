---
type: handoff
title: "NLM-KN Deployment Closure Handoff — 2026-08-03"
description: "Current priority and execution checklist for closing Linode and PVE2 deployment evidence before RWDKN semantic-model and remediation-loop work."
tags: [handoff, nlmkn, deployment, linode, pve2, jenkins, evidence, rwdkn, semantic-model]
version: 1.0.0
created: 2026-08-03
updated: 2026-08-03
links:
  - "2026-07-29-nlmkn-deployment-targets-handoff.md"
  - "../../nlmkn/docs/deployment-configuration.md"
  - "../../nlmkn/deploy/targets/localhost.json"
  - "../../nlmkn/deploy/targets/linode.json"
  - "../../nlmkn/Jenkinsfile"
  - "../../rwdkn-data-pipeline/docs/specs/near-future-improvement-dataset-remediation-looping.md"
  - "../../rwdkn-data-pipeline/docs/specs/nlm-kn-requirements-traceability-matrix.md"
---

# NLM-KN Deployment Closure Handoff — 2026-08-03

**Prepared:** 2026-08-03 17:23 EDT  
**Workspace:** `/home/chandrakola/development/NLM/kg`  
**Primary repository:** `/home/chandrakola/development/NLM/kg/nlmkn`  
**Documentation repository:** `/home/chandrakola/development/NLM/kg/kg-docs`  
**Priority decision:** Finish the bounded deployment work before starting RWDKN semantic-model implementation or dataset-remediation-loop implementation.

## Decision and sequencing

The work order is now:

1. Close Linode deployment verification and evidence.
2. Define, deploy, and verify the PVE2 target using the same config-driven path.
3. Publish a dated deployment completion report.
4. Begin the RWDKN LinkML/RDF/OWL/SHACL semantic-model track.
5. Begin the DailyMed dataset-remediation-loop pilot only after deployment reproducibility and initial semantic validation rules exist.

OpenClaw, Hermes, DSPy, external agent-memory systems, and similar agent-platform experiments are deferred. They are not prerequisites for NLM-KN deployment or the RWDKN data model.

## Starting evidence

The prior handoff, `2026-07-29-nlmkn-deployment-targets-handoff.md`, recorded the following as completed at that time:

- Config-driven deployment targets were published on `nlmkn/main` at commit `5b4979d6cdc5fd4a9915200dc0db0e79b9f7bd03`.
- The SSO routing fix was published at `d13f123`.
- `localhost.json` and `linode.json` were checked-in non-secret deployment descriptors.
- Deployment configuration unit tests, localhost registry rendering, shell syntax, Compose rendering, and `git diff --check` passed.
- `nlmkn/docs/reports/local-smoke.md` remained a deliberately unstaged local evidence change.

These facts are historical starting evidence, not confirmation of current runtime state. Revalidate the branch, Jenkins jobs, hosts, routes, secrets handling, and live services before marking any item below complete.

## P0 — Establish current state

- [ ] Confirm `nlmkn`, `shared-infra`, `rwdkn-service`, and participant repositories are on the intended branches and record their current Git SHAs.
- [ ] Confirm Jenkins has fetched the intended `nlmkn/main` revision.
- [ ] Confirm whether the Linode target VM and associated DNS/tunnel/Traefik stack are currently running.
- [ ] Inspect the generated `nlmkn/docs/reports/local-smoke.md` change and either publish it as dated evidence or discard it deliberately.
- [ ] Record current Jenkins job names, last build numbers, results, and image tags before launching a new deployment.

## P1 — Validate Linode through Jenkins

- [ ] Run the NLM-KN Jenkins job with `DEPLOYMENT_ID=linode` and explicitly selected `SCOPE` and `CLEAN` values.
- [ ] Confirm the load-deployment stage resolves the expected target, protocol, and NLM-KN hostname from `deploy/targets/linode.json`.
- [ ] Confirm the rendered remote `deploy/deployment.env` contains no secret values.
- [ ] Confirm `/run/nlmkn/sso.env` remains mode `0640` and contains only the required runtime SSO credentials.
- [ ] Confirm Traefik registers Hub, health, and `/__auth` routes for NLM-KN and every participant hostname.
- [ ] Verify the NLM-KN homepage and Hub render in a real browser.
- [ ] Verify unauthenticated redirect, login, authenticated registry access, and shared SSO behavior.
- [ ] Verify participant health, manifest, capabilities, entity, neighborhood, and query routes.
- [ ] Specifically retest the RWDKN explorer redirect and confirm its pages load data rather than merely returning HTTP 200.
- [ ] Run the federation smoke suite and save a dated report.
- [ ] Record the deployed image tags, Git SHAs, target descriptor, Jenkins build URLs/numbers, and smoke result.

## P1 — Define and validate PVE2

- [ ] Decide whether PVE2 is internal-only, public, or dual-access.
- [ ] Record the SSH or Tailscale hostname Jenkins will use.
- [ ] Choose the DNS suffix and hostnames for NLM-KN, RWDKN, Browser, CKN, GKN, and HPHYSKN.
- [ ] Decide HTTP versus HTTPS and define the valid shared SSO cookie domain.
- [ ] Create `nlmkn/deploy/targets/pve2.json` from the deployment descriptor contract, not from Jenkins conditionals.
- [ ] Create or select the PVE2 Infisical environment and verify Jenkins can read only the intended secrets.
- [ ] Provision or verify DNS, Traefik, Docker, and the external `nlmkn-edge` network on PVE2.
- [ ] Validate `pve2.json` with `scripts/deployment_config.py` and render every Compose configuration before deployment.
- [ ] Run Jenkins with `DEPLOYMENT_ID=pve2` without adding a PVE2-specific Jenkinsfile branch or application-code path.
- [ ] Repeat the browser, SSO, participant-route, RWDKN explorer, and federation smoke verification performed for Linode.
- [ ] Record PVE2 image tags, Git SHAs, route matrix, Jenkins build evidence, and smoke results.

## P1 — Close deployment evidence

- [ ] Produce a dated completion report covering Linode and PVE2.
- [ ] Include target descriptor revisions, repository SHAs, image tags, Jenkins build identifiers, route matrices, smoke reports, and known exceptions.
- [ ] Confirm no secrets appear in Git, generated descriptors, reports, or Jenkins logs.
- [ ] Document rollback to the previous known-good deployment and verify the rollback inputs are still available.
- [ ] Update the July 29 handoff or link it to the completion report so reviewers can follow the evidence chain.

## Deployment exit criteria

Deployment closure is complete only when:

1. `DEPLOYMENT_ID=linode` drives a successful deployment from the checked-in descriptor.
2. Linode homepage, SSO, participant routes, RWDKN explorer, and federation smoke checks pass with saved evidence.
3. A valid `pve2.json` drives the same Jenkinsfile without deployment-target-specific application changes.
4. PVE2 routing, SSO, participant APIs, explorer behavior, and federation checks pass with saved evidence.
5. Git SHAs, images, Jenkins builds, configuration inputs, and smoke outputs are correlated for both targets.
6. No secret value is committed or exposed in deployment evidence.
7. Rollback is documented against a known-good version.

## Next workstream — RWDKN semantic model

After deployment closure, begin an `RWDKN Semantic Data Model v0.1` covering:

- prioritized competency questions;
- a BioLink-aligned LinkML source schema;
- RDF/OWL semantics and imports from authoritative biomedical ontologies;
- SHACL validation shapes;
- SSSOM terminology mappings;
- evidence, provenance, qualifier, and curation semantics;
- deterministic projections to KGX/Biolink, JSON-LD, RDF/Turtle, and Neo4j LPG; and
- automated schema and conformance fixtures.

The canonical model must not be defined solely by the current Neo4j layout. OWL expresses meaning; LinkML and SHACL provide structural and instance validation; KGX and Neo4j remain interoperable projections.

## Deferred workstream — dataset remediation loop

Do not implement `rwdkn-data-pipeline/docs/specs/near-future-improvement-dataset-remediation-looping.md` until:

- the deployment exit criteria above are satisfied;
- local and remote promotion behavior is reproducible;
- failed stages cannot publish partial KGX as current;
- run, code, prompt, model, resolver, schema, and configuration provenance are recorded; and
- initial LinkML/SHACL rules provide an authoritative definition of semantic conformance.

When authorized, start with a bounded DailyMed pilot. Preserve immutable rounds, replay deltas rather than entire datasets, route failures to the owning stage, cap retries, preserve original evidence, and publish only a consolidated validated snapshot.

## Immediate next action

Perform the P0 current-state audit. Do not launch a clean or destructive deployment until the current Linode/PVE2 resources, Jenkins revisions, saved evidence, and rollback inputs have been identified.
