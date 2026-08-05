---
type: evidence
title: "NLM-KN PVE2 Deployment Evidence — 2026-08-04"
description: "Correlated deployment, runtime, federation, graph, routing, and rollback evidence for the public Proxmox PVE2 target."
tags: [evidence, nlmkn, pve2, proxmox, cloudflare, jenkins, neo4j, federation]
version: 1.1.0
created: 2026-08-04
updated: 2026-08-04
links:
  - "2026-08-03-nlmkn-deployment-closure-handoff.md"
  - "../../nlmkn/deploy/targets/pve2.json"
  - "../../nlmkn/docs/reports/phase-2_5-2026-08-04.md"
  - "../../nlmkn/docs/reports/phase-2_5-2026-08-03.md"
---

# NLM-KN PVE2 Deployment Evidence — 2026-08-04

**Evidence captured:** 2026-08-04 21:27–21:50 EDT / 2026-08-05 01:27–01:50 UTC
**Target:** `pve2`  
**Provider:** Proxmox  
**Jenkins SSH target:** `nlmkn-pve2-2`  
**Pulumi stack:** `pve2`  
**Public Hub:** `https://nlmkn-pve2.kolac.us`  
**Overall result:** **PASS**

The PVE2 VM, Cloudflare ingress, shared SSO, Hub, four federation participants, RWDKN
application, Neo4j Browser, and Neo4j graph are running. After correcting provider-specific
Browser parameter propagation, Jenkins coordinator build 89 and RWDKN build 10 completed
successfully, and federation evidence build 39 passed all eight automated gates. The operator
separately reported successful application and SSO testing.

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
| `shared-infra` | `2927737aad63151322de545673bf412e6caaee1a` for successful foundation build 68; routing fix `a5fd481d3c9a41be047d31a32e66a2bcf6b5ef66` | Jenkins checkout and `main` |
| `nlmkn` | `4b3099f5270a7d4d2fe1363f69359514ac9be74d` | Coordinator and evidence builds |
| `rwdkn-service` | `c3dd1ee5f45faf4cb7627d69f43314506667d727` | RWDKN pipeline build 10 and image tags |
| `ckn` | `bf3e030be1d3f1507a7c9c41d1405acbca25f332` | CKN build 63 |
| `gkn` | `431fb94cc4776677d8131fcadf4d00b5e7ded201` | GKN build 62 |
| `hphyskn` | `d6e75d2dbe17453a54e0423199ad203aec421282` | HPHYSKN build 61 |
| RWDKN data artifact | `20260707` | Coordinator build 89 parameter; data was not reloaded |

## Jenkins evidence

| Job | Build | Started UTC | Result | Purpose |
|---|---:|---|---|---|
| `shared-infra-deploy` | [68](https://jenkins.kolac.us/job/shared-infra-deploy/68/) | 2026-08-04T18:23:34Z | SUCCESS | Proxmox/Pulumi `up`, Cloudflare DNS, ingress reachability |
| `nlmkn-coordinator` | [89](https://jenkins.kolac.us/job/nlmkn-coordinator/89/) | 2026-08-05T01:45:34Z | SUCCESS | Corrected PVE2 RWDKN refresh and evidence orchestration |
| `ckn-deploy` | [63](https://jenkins.kolac.us/job/ckn-deploy/63/) | 2026-08-04T18:35:29Z | SUCCESS | CKN deployment |
| `gkn-deploy` | [62](https://jenkins.kolac.us/job/gkn-deploy/62/) | 2026-08-04T18:35:29Z | SUCCESS | GKN deployment |
| `hphyskn-deploy` | [61](https://jenkins.kolac.us/job/hphyskn-deploy/61/) | 2026-08-04T18:35:29Z | SUCCESS | HPHYSKN deployment |
| `rwdkn/pipeline` | [10](https://jenkins.kolac.us/job/rwdkn/job/pipeline/10/) | 2026-08-05T01:46:09Z | SUCCESS | RWDKN deployment with PVE2 Browser parameters |
| `hub-deploy` | [76](https://jenkins.kolac.us/job/hub-deploy/76/) | 2026-08-04T18:35:29Z | SUCCESS | Hub and SSO deployment |
| `federation-smoke-test` | [39](https://jenkins.kolac.us/job/federation-smoke-test/39/) | 2026-08-05T01:47:29Z | SUCCESS | Post-correction eight-gate federation evidence |

Coordinator build 89 used `DEPLOYMENT_ID=pve2`, `DEPLOY_TYPE=refresh`,
`REFRESH_SCOPE=application`, `RUN_EVIDENCE=true`, and `ARTIFACT_VERSION=20260707`.
Evidence build 39 used `HUB_URL=https://nlmkn-pve2.kolac.us`,
`FTH=nlmkn-pve2-2`, and `NO_SSH=false`.

## Runtime container inventory

Captured over the tailnet from `nlmkn-pve2-2` after the corrected deployment at
2026-08-05T01:48:57Z.

| Container | Image | Runtime state |
|---|---|---|
| `nlmkn-auth` | `chandrakola/nlmkn-auth:108d5c9` | Up, healthy |
| `nlmkn-hub` | `nginx:alpine` | Up |
| `nlmkn-traefik` | `traefik:v3` | Up |
| `nlmkn-cloudflared` | `cloudflare/cloudflared:latest` | Up |
| `kn-ckn` | `nginx:alpine` | Up |
| `kn-gkn` | `nginx:alpine` | Up |
| `kn-hphyskn` | `nginx:alpine` | Up |
| `rwdkn-query-app` | `chandrakola/rwdkn-query-app:c3dd1ee` | Up, healthy |
| `rwdkn-sentence-api` | `chandrakola/rwdkn-sentence-api:c3dd1ee` | Up, healthy |
| `rwdkn-ground-truth` | `chandrakola/rwdkn-ground-truth:c3dd1ee` | Up, healthy |
| `rwdkn-insight-studio` | `chandrakola/rwdkn-insight-studio:c3dd1ee` | Up, healthy |
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
| `https://browser-pve2.kolac.us/` with browser navigation headers | HTTP 302 to `/browser/` | PASS |
| `https://browser-pve2.kolac.us/browser/` | HTTP 200; HTML title `Neo4j Browser` | PASS |
| `https://browser-pve2.kolac.us/` discovery request | HTTP 200 JSON; Bolt host `browser-pve2.kolac.us:443` | PASS |
| `wss://browser-pve2.kolac.us/` upgrade | HTTP 101 Switching Protocols through Cloudflare | PASS |

Jenkins evidence authenticated the Hub registry and exercised all four participant query
contracts. The operator reported successful real-browser application and shared-SSO behavior.
The in-app browser-control surface was unavailable, so the signed-in visual acceptance remains
operator-provided evidence. Independent public checks verified the Neo4j Browser HTML shell,
discovery document, advertised Bolt address, and WebSocket upgrade path without using a database
credential.

## Federation evidence

The durable machine-generated report is
`nlmkn/docs/reports/phase-2_5-2026-08-04.md`, refreshed from federation-smoke-test build 39.

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

## Resolved Browser hostname exception

The initial deployment left `BROWSER_HOST=browser.kolac.us` in the RWDKN runtime because the
coordinator did not pass provider-specific Hub/Browser routing values to `rwdkn/pipeline`.

The generic correction was published as:

- `shared-infra` `a5fd481` — derive and pass `HUB_HOST`, `BROWSER_HOST`, public protocol, and
  `NEO4J_BOLT_ADVERTISED_ADDRESS` from the selected deployment descriptor; and
- `rwdkn-service` `c3dd1ee` — accept those values as pipeline parameters and export them to the
  runtime renderer.

RWDKN build 9 safely primed Jenkins' declarative parameter schema. Coordinator build 89 then
passed the PVE2 values to RWDKN build 10. The running environment now reports
`BROWSER_HOST=browser-pve2.kolac.us`, and Neo4j advertises
`browser-pve2.kolac.us:443`. Browser UI, discovery, WebSocket, container-health, graph-count, and
eight-gate federation checks all passed after the correction.

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

The public PVE2 NLM-KN application, federation, Neo4j Browser, and graph are deployed and
operating successfully. All declared PVE2 hostnames align with the running configuration, 8/8
automated federation gates are green, the public Browser WebSocket path upgrades successfully,
and the graph data remained at 6,926 nodes and 17,473 edges after the application-only refresh.
