---
type: evidence
title: "NLM-KN PVE2 Deployment Evidence — 2026-08-04"
description: "Correlated deployment, runtime, federation, graph, routing, and rollback evidence for the public Proxmox PVE2 target."
tags: [evidence, nlmkn, pve2, proxmox, cloudflare, jenkins, neo4j, federation]
version: 1.0.0
created: 2026-08-04
updated: 2026-08-04
links:
  - "2026-08-03-nlmkn-deployment-closure-handoff.md"
  - "../../nlmkn/deploy/targets/pve2.json"
  - "../../nlmkn/docs/reports/phase-2_5-2026-08-04.md"
  - "../../nlmkn/docs/reports/phase-2_5-2026-08-03.md"
---

# NLM-KN PVE2 Deployment Evidence — 2026-08-04

**Evidence captured:** 2026-08-04 21:27–21:33 EDT / 2026-08-05 01:27–01:33 UTC  
**Target:** `pve2`  
**Provider:** Proxmox  
**Jenkins SSH target:** `nlmkn-pve2-2`  
**Pulumi stack:** `pve2`  
**Public Hub:** `https://nlmkn-pve2.kolac.us`  
**Overall result:** **PASS WITH ONE KNOWN CONFIGURATION EXCEPTION**

The PVE2 VM, Cloudflare ingress, shared SSO, Hub, four federation participants, RWDKN
application, and Neo4j graph are running. Jenkins coordinator build 87 completed successfully,
and federation evidence build 37 passed all eight automated gates. The operator separately
reported successful application and SSO testing.

The declared PVE2 Neo4j Browser hostname is not active in the running RWDKN configuration.
`browser-pve2.kolac.us` returns HTTP 404 because the container retained
`BROWSER_HOST=browser.kolac.us`. This exception does not invalidate the Hub, participant API,
federation, explorer, or graph evidence below, but it must be corrected before declaring every
descriptor route aligned with runtime.

## Configuration identity

The checked-in descriptor `nlmkn/deploy/targets/pve2.json` declares:

| Field | Value |
|---|---|
| Target type | `ssh-compose` |
| SSH host | `nlmkn-pve2-2` |
| Foundation provider | `proxmox` |
| Pulumi stack | `pve2` |
| Public scheme | `https` |
| Cookie domain | `.kolac.us` |
| Infisical environment | `pve2` |

No runtime secret value is stored in the descriptor or this report.

## Repository and release revisions

| Component | Revision used by evidence/deployment | Evidence |
|---|---|---|
| `shared-infra` | `2927737aad63151322de545673bf412e6caaee1a` for successful foundation build 68; current completion revision `349df22e7f1ced4ee7babc14e20263eb060b0b31` | Jenkins checkout and local `main` |
| `nlmkn` | `108d5c9c702fba5761e961eb3688ebff3742ef6f` | Coordinator, Hub, and evidence builds |
| `rwdkn-service` | `80ee6c663fb85993f785d20c4549d8f9808350d9` | RWDKN pipeline build 8 and image tags |
| `ckn` | `bf3e030be1d3f1507a7c9c41d1405acbca25f332` | CKN build 63 |
| `gkn` | `431fb94cc4776677d8131fcadf4d00b5e7ded201` | GKN build 62 |
| `hphyskn` | `d6e75d2dbe17453a54e0423199ad203aec421282` | HPHYSKN build 61 |
| RWDKN data artifact | `20260707` | Coordinator build 87 parameter |

## Jenkins evidence

| Job | Build | Started UTC | Result | Purpose |
|---|---:|---|---|---|
| `shared-infra-deploy` | [68](https://jenkins.kolac.us/job/shared-infra-deploy/68/) | 2026-08-04T18:23:34Z | SUCCESS | Proxmox/Pulumi `up`, Cloudflare DNS, ingress reachability |
| `nlmkn-coordinator` | [87](https://jenkins.kolac.us/job/nlmkn-coordinator/87/) | 2026-08-04T18:34:59Z | SUCCESS | PVE2 application refresh and evidence orchestration |
| `ckn-deploy` | [63](https://jenkins.kolac.us/job/ckn-deploy/63/) | 2026-08-04T18:35:29Z | SUCCESS | CKN deployment |
| `gkn-deploy` | [62](https://jenkins.kolac.us/job/gkn-deploy/62/) | 2026-08-04T18:35:29Z | SUCCESS | GKN deployment |
| `hphyskn-deploy` | [61](https://jenkins.kolac.us/job/hphyskn-deploy/61/) | 2026-08-04T18:35:29Z | SUCCESS | HPHYSKN deployment |
| `rwdkn/pipeline` | [8](https://jenkins.kolac.us/job/rwdkn/job/pipeline/8/) | 2026-08-04T18:35:29Z | SUCCESS | RWDKN application deployment |
| `hub-deploy` | [76](https://jenkins.kolac.us/job/hub-deploy/76/) | 2026-08-04T18:35:29Z | SUCCESS | Hub and SSO deployment |
| `federation-smoke-test` | [37](https://jenkins.kolac.us/job/federation-smoke-test/37/) | 2026-08-04T18:36:44Z | SUCCESS | Eight-gate live federation evidence |

Coordinator build 87 used `DEPLOYMENT_ID=pve2`, `DEPLOY_TYPE=refresh`,
`REFRESH_SCOPE=application`, `RUN_EVIDENCE=true`, and `ARTIFACT_VERSION=20260707`.
Evidence build 37 used `HUB_URL=https://nlmkn-pve2.kolac.us`,
`FTH=nlmkn-pve2-2`, and `NO_SSH=false`.

## Runtime container inventory

Captured over the tailnet from `nlmkn-pve2-2` at 2026-08-05T01:27:51Z.

| Container | Image | Runtime state |
|---|---|---|
| `nlmkn-auth` | `chandrakola/nlmkn-auth:108d5c9` | Up, healthy |
| `nlmkn-hub` | `nginx:alpine` | Up |
| `nlmkn-traefik` | `traefik:v3` | Up |
| `nlmkn-cloudflared` | `cloudflare/cloudflared:latest` | Up |
| `kn-ckn` | `nginx:alpine` | Up |
| `kn-gkn` | `nginx:alpine` | Up |
| `kn-hphyskn` | `nginx:alpine` | Up |
| `rwdkn-query-app` | `chandrakola/rwdkn-query-app:80ee6c6` | Up, healthy |
| `rwdkn-sentence-api` | `chandrakola/rwdkn-sentence-api:80ee6c6` | Up, healthy |
| `rwdkn-ground-truth` | `chandrakola/rwdkn-ground-truth:80ee6c6` | Up, healthy |
| `rwdkn-insight-studio` | `chandrakola/rwdkn-insight-studio:80ee6c6` | Up, healthy |
| `rwdkn-edge` | `nginx:alpine` | Up |
| `rwdkn-neo4j` | `neo4j:5-community` | Up, healthy |
| `rwdkn-postgres` | `pgvector/pgvector:pg16` | Up, healthy |
| `rwdkn-redis` | `redis:7-alpine` | Up, healthy |

The external `nlmkn-edge` bridge network was present with eight attached containers.

## Public route and authentication matrix

Unauthenticated HTTP evidence was captured directly from the public endpoints.

| Route | Observed | Assessment |
|---|---|---|
| `https://nlmkn-pve2.kolac.us/health` | HTTP 200 | PASS |
| `https://nlmkn-pve2.kolac.us/` | HTTP 302 to same-origin HTTPS `/__auth/login` | PASS |
| `https://nlmkn-pve2.kolac.us/hub/` | HTTP 302 to same-origin HTTPS `/__auth/login` | PASS |
| `https://rwdkn-pve2.kolac.us/` | HTTP 200 | PASS; public participant landing behavior |
| `https://rwdkn-pve2.kolac.us/explorer/` | HTTP 302 to same-origin HTTPS `/__auth/login` | PASS |
| RWDKN Insight, Sentences, Triples, and NLQ explorer paths | HTTP 302 to same-origin HTTPS `/__auth/login` | PASS |
| CKN `/api/health` | HTTP 200 | PASS |
| GKN `/api/health` | HTTP 200 | PASS |
| HPHYSKN `/api/health` | HTTP 200 | PASS |
| RWDKN `/api/health` | HTTP 200 | PASS |
| `https://browser-pve2.kolac.us/` | HTTP 404 | **KNOWN EXCEPTION** |

Jenkins evidence authenticated the Hub registry and exercised all four participant query
contracts. The operator reported successful real-browser application and shared-SSO behavior.
This evidence session did not copy runtime credentials into an automated browser, and the
in-app browser-control surface was unavailable; therefore that manual acceptance is identified
as operator-provided evidence rather than independently repeated visual evidence.

## Federation evidence

The durable machine-generated report is
`nlmkn/docs/reports/phase-2_5-2026-08-04.md`, copied from federation-smoke-test build 37.

| Gate | Result |
|---|---|
| Registry shape | PASS |
| CORS preflight | PASS |
| Query contract | PASS |
| Schema conformance | PASS |
| Runtime federation | PASS |
| Degraded mode | PASS |
| Shared-route safety | PASS |
| Sensitive-response scan | PASS |

The live query returned results from all four participants: RWDKN 85, GKN 3, CKN 1, and
HPHYSKN 1.

## Neo4j graph evidence

Captured directly inside the healthy `rwdkn-neo4j` container without printing the database
credential.

| Measure | Count |
|---|---:|
| All database nodes | 6,926 |
| All database edges | 17,473 |
| `KGXNode` nodes | 6,926 |
| Relationships between `KGXNode` nodes | 17,473 |
| Nodes without a source property | 0 |
| Edges without a source property | 0 |

### Counts by knowledge source

| Knowledge source | Node memberships | Edges |
|---|---:|---:|
| `clinical-findings` | 504 | 1,243 |
| `dailymed` | 90 | 504 |
| `faers` | 5,999 | 12,876 |
| `pubmed` | 716 | 2,791 |
| `real-kg:study-scope` | 1 | 14 |
| `rxnav` | 28 | 45 |

Node membership counts are not additive: a normalized concept can belong to multiple sources,
so the membership total can exceed the 6,926 unique nodes. Edge counts are grouped by each
relationship's scalar `knowledge_source`.

## Secrets and evidence handling

- No password, Infisical access token, Cloudflare tunnel token, database URI, or cookie value
  was copied into this report.
- Runtime database queries expanded the container credential only inside the remote process and
  returned counts only.
- The federation security scan passed with no private or sensitive marker in collected responses.
- Container environment output was limited to non-secret routing keys while investigating the
  Browser hostname exception.

## Known exception and corrective action

The descriptor declares `BROWSER_HOST=browser-pve2.kolac.us`, but the deployed `rwdkn-edge`
container reports `BROWSER_HOST=browser.kolac.us`. The coordinator passes `DEPLOY_HOST`,
`RWDKN_HOST`, and `HUB_ORIGIN` to `rwdkn/pipeline`, but it does not pass `BROWSER_HOST` or
`NEO4J_BOLT_ADVERTISED_ADDRESS`. The RWDKN runtime renderer therefore applies its Linode-era
defaults.

Corrective action:

1. Add `BROWSER_HOST` and `NEO4J_BOLT_ADVERTISED_ADDRESS` as RWDKN pipeline parameters.
2. Pass the PVE2 descriptor values from the coordinator.
3. Run an application-only PVE2 refresh.
4. Verify `browser-pve2.kolac.us` discovery, Browser UI, and Bolt-over-WebSocket behavior.
5. Append the successful build and route evidence to this report.

## Rollback

The current known-good application inputs are the revisions and immutable application image
tags recorded above. A rollback was documented but not executed because that would disrupt the
working deployment.

Application rollback procedure:

1. Revert the faulty change on the affected repository's `main` branch so history remains
   auditable; do not move or force-push `main`.
2. Confirm the previously recorded image tags remain available in Docker Hub.
3. Run `nlmkn-coordinator` with `DEPLOYMENT_ID=pve2`, `DEPLOY_TYPE=refresh`,
   `REFRESH_SCOPE=application`, `CLEAN_OPTION=none`, and `RUN_EVIDENCE=true`.
4. Confirm container health and repeat the public route and eight-gate federation checks.

Foundation recovery procedure:

1. Use the existing Proxmox Pulumi stack `pve2`; run `shared-infra-deploy ACTION=preview` first.
2. Do not run `destroy` or a volume wipe as an application rollback.
3. Preserve the Neo4j, PostgreSQL, and Ground Truth volumes before any authorized destructive
   recovery.
4. If infrastructure reconciliation is necessary, run the checked-in provider path and verify
   the existing VM target before approving any replacement.

## Conclusion

The public PVE2 NLM-KN application and federation are deployed and operating successfully, with
8/8 automated federation gates green and the graph data present. Deployment closure remains
conditional on aligning the PVE2-specific Neo4j Browser hostname with the checked-in descriptor
and appending that route's successful evidence.
