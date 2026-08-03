---
type: handoff
title: "NLM-KN Deployment Closure Handoff — 2026-08-03"
description: "Current priority and execution checklist for closing Linode and PVE2 deployment evidence before RWDKN semantic-model and remediation-loop work."
tags: [handoff, nlmkn, deployment, linode, pve2, jenkins, evidence, rwdkn, semantic-model]
version: 1.2.0
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

- [x] Confirm `nlmkn`, `shared-infra`, `rwdkn-service`, and participant repository branches and record their current Git SHAs. See the audit snapshot below; branch intent for `shared-infra` still requires closure because Jenkins tracks a feature branch rather than `main`.
- [x] Confirm the NLM-KN revision used by the latest successful Hub deployment. It used `83f221c`, which predates target-descriptor commit `5b4979d` and current `nlmkn/main` at `e270ba0`; a fresh deployment is required.
- [x] Confirm current Linode VM, DNS, tunnel, Traefik, and application state. The environment is in a partial-teardown state; details are below.
- [x] Inspect `nlmkn/docs/reports/local-smoke.md`. It is committed on `nlmkn/main` at `e270ba0`, records an eight-of-eight localhost pass generated 2026-07-28, and skipped the two SSH-dependent checks because it ran with `--no-ssh`.
- [x] Record current Jenkins job status before launching a new deployment. Coordinator build 81 and shared-infra build 45 failed; their last successful builds were 80 and 44 respectively.

## Audit snapshot — 2026-08-03 17:53 EDT

### Repository and Jenkins revisions

| Component | Branch | Current revision | Jenkins observation |
|---|---|---|---|
| `kg-docs` | `main` | `e3e0be8` | Local branch is one commit ahead of `origin/main`; this handoff is not pushed yet. |
| `nlmkn` | `main` | `e270ba0` | Latest successful Hub build 69 used older revision `83f221c`. |
| `shared-infra` | `appmod/java-upgrade-20260722110312` | `00ee477` plus uncommitted Jenkins safety guards | Coordinator and shared-infra jobs track this feature branch; `main` is at `bb36024`. |
| `rwdkn-service` | `main` | `80ee6c6` | Current repository snapshot recorded; no live Linode service exists. |
| `rwdkn-data-pipeline` | `main` | `c44fa7` | Current repository snapshot recorded; no live Linode service exists. |
| `ckn` | `main` | `bf3e030` | Current repository snapshot recorded; no live Linode service exists. |
| `gkn` | `main` | `431fb94` | Current repository snapshot recorded; no live Linode service exists. |
| `hphyskn` | `main` | `d6e75d2` | Current repository snapshot recorded; no live Linode service exists. |

### Linode state

- The target remains reachable through Tailscale as `real-kg-demo` and resolves to the expected Linode VM.
- Only `nlmkn-traefik` and `nlmkn-cloudflared` are running remotely; the Hub, auth service, RWDKN, CKN, GKN, and HPHYSKN are absent.
- All six `*.kolac.us` deployment hostnames fail public DNS resolution.
- The latest coordinator run, build 81, requested `DEPLOY_TYPE=teardown` with `TARGET_ENV=localhost` but incorrectly invoked the cloud foundation destroy job with `DEPLOY_HOST=localhost`.
- Shared-infra build 45 deleted the six Cloudflare DNS records, then failed to delete the tunnel configuration because the remote tunnel remained connected. The result is a partial teardown, not a valid deployed or cleanly destroyed state.
- A diagnostic exposed a tunnel credential in transient command output. Do not copy it into reports or logs; rotate the credential during foundation recovery.

### Safety correction in progress

- `shared-infra/Jenkinsfile.coordinator` now refuses a localhost teardown, skips the cloud-foundation stage for localhost full deployments, and pins cloud foundation actions to `real-kg-demo`.
- `shared-infra/Jenkinsfile` independently refuses `up` and `destroy` when `DEPLOY_HOST=localhost`.
- `git diff --check` passes for both changes. They remain uncommitted and unpushed pending review and Jenkins publication.
- Before rebuilding Linode, stop the connected remote tunnel, complete or reconcile the failed foundation teardown, rotate the tunnel credential, and then deploy from the current descriptor-aware NLM-KN revision.

## Linode recovery result — 2026-08-03 18:46 EDT

- Shared-infra build 46 completed the failed teardown and deleted the four remaining Pulumi resources.
- Shared-infra build 47 created a replacement Linode VM, a new Cloudflare tunnel, and all six DNS records. It failed only because the retired Tailscale device temporarily retained the configured `real-kg-demo` name.
- After deleting the retired Tailscale device and assigning the replacement the canonical name, ingress build 48 succeeded. Idempotence build 49 also succeeded.
- The tunnel replacement rotated the credential exposed during the initial diagnostic.
- Jenkins safety fixes were published on the Jenkins-tracked `shared-infra` branch at `2aeda17`; explicit Hub descriptor propagation was published at `96bbaea`.
- CKN build 60, GKN build 59, HPHYSKN build 58, RWDKN pipeline build 5, and final Hub build 73 succeeded.
- Coordinator build 82 is recorded as failed because its first Hub child, build 70, ran while Jenkins still had the pre-descriptor parameter schema. Hub build 71 proved the explicit `DEPLOYMENT_ID=linode` retry, and builds 72–73 deployed the registry and HTTPS redirect corrections.
- `nlmkn` fixes were published at `b40d997` (export descriptor values for registry rendering), `eef36a7` (case-insensitive evidence header handling), and `b0a0648` (preserve public HTTPS in forward-auth redirects).
- Final federation evidence build 33 passed all eight gates against deployed commit `b0a0648`. The durable report is `nlmkn/docs/reports/phase-2_5-2026-08-03.md`, published in documentation commit `8600275`.
- A secret-safe live login returned an HTTPS redirect to `/hub/`, and the resulting cookie accessed `/hub/api/registry` with HTTP 200.
- Remote `/run/nlmkn/sso.env` is mode `0640`, owned by `root:root`, and contains only `SSO_USER` and `SSO_PASSWORD`. The rendered `deploy/deployment.env` contains deployment configuration only and no secret-like values.
- Public `https://nlmkn.kolac.us/` now redirects to the HTTPS login URL, and `/health` returns HTTP 200.
- An isolated browser followed `https://rwdkn.kolac.us` to `https://rwdkn.kolac.us/__auth/login?rd=https%3A%2F%2Frwdkn.kolac.us%2Fexplorer%2F` and rendered the `NLM-KN — Sign in` page. Authenticated explorer data rendering remains a manual follow-up because no runtime credential was copied into browser state.

### Final container/image inventory

| Containers | Image/tag | Runtime result |
|---|---|---|
| `nlmkn-auth` | `chandrakola/nlmkn-auth:b0a0648` | Healthy |
| `nlmkn-hub` | `nginx:alpine` | Running |
| `nlmkn-traefik` | `traefik:v3` | Running |
| `nlmkn-cloudflared` | `cloudflare/cloudflared:latest` | Running |
| `kn-ckn`, `kn-gkn`, `kn-hphyskn` | `nginx:alpine` | Running; federation checks passed |
| `rwdkn-ground-truth`, `rwdkn-insight-studio`, `rwdkn-query-app`, `rwdkn-sentence-api` | `chandrakola/*:80ee6c6` | Healthy |
| `rwdkn-edge` | `nginx:alpine` | Running |
| `rwdkn-neo4j` | `neo4j:5-community` | Healthy |
| `rwdkn-postgres` | `pgvector/pgvector:pg16` | Healthy |
| `rwdkn-redis` | `redis:7-alpine` | Healthy |

## P1 — Validate Linode through Jenkins

- [x] Run the NLM-KN Jenkins job with `DEPLOYMENT_ID=linode`, `SCOPE=both`, and `CLEAN=none`.
- [x] Confirm the load-deployment stage resolves `linode -> real-kg-demo (https://nlmkn.kolac.us)` from `deploy/targets/linode.json`.
- [x] Confirm the rendered remote `deploy/deployment.env` contains no secret values.
- [x] Confirm `/run/nlmkn/sso.env` remains mode `0640` and contains only the required runtime SSO credentials.
- [x] Confirm the public Hub, health, `/__auth`, and participant routes resolve through the rebuilt tunnel and Traefik.
- [ ] Verify the NLM-KN homepage and Hub render in a real browser.
- [x] Verify unauthenticated HTTPS redirect, credential login, cookie creation, and authenticated registry access without exposing the runtime credential.
- [ ] Verify participant health, manifest, capabilities, entity, neighborhood, and query routes.
- [ ] Specifically retest the RWDKN explorer redirect and confirm its pages load data rather than merely returning HTTP 200.
- [x] Run the federation smoke suite with SSH checks and save the dated eight-of-eight report.
- [x] Record deployed image tags, Git SHAs, target descriptor, Jenkins build numbers, and the final smoke result in this handoff.

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

Finish the remaining Linode browser/explorer route checks and capture deployed image tags. Then define `pve2.json` only after the PVE2 access model, Jenkins SSH/Tailscale hostname, DNS suffix, TLS mode, cookie domain, and Infisical environment are known.
