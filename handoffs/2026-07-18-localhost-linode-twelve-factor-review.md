---
type: handoff
title: "Localhost and Linode Deployment Review Handoff"
description: "Tomorrow's cleanup and verification plan for localhost, public Linode deployment, and Twelve-Factor configuration independence across NLM-KN."
tags: [handoff, localhost, linode, deployment, twelve-factor, configuration, nlmkn]
version: 1.0.0
created: 2026-07-18
updated: 2026-07-18
links:
  - "../architecture/twelve-factor-compliance.md"
  - "../architecture/repository-split.md"
  - "../../nlmkn/scripts/run_local_stack.sh"
  - "../../rwdkn-service/deploy/docker-compose.nlmkn-local.yml"
  - "../../rwdkn-service/deploy/rwdkn/docker-compose.rwdkn.yml"
---

# Localhost and Linode Deployment Review Handoff

**Prepared:** 2026-07-18 00:13 EDT
**For:** Next work session
**Primary goal:** Clean and review both localhost and public-cloud deployment, then verify that application code is independent of environment-specific deployment configuration.

## 1. Current verified state

The following repositories were clean and synchronized with `origin/main` when this handoff was prepared:

- `nlmkn`
- `rwdkn-service`
- `rwdkn-data-pipeline`
- `ckn`
- `gkn`
- `hphyskn`
- `shared-infra`

The localhost stack was running successfully:

| Surface | Verified behavior |
|---|---|
| `http://nlmkn.localhost/` | SSO redirect, then authenticated Hub at `/hub/` |
| `http://rwdkn.localhost/api/health` | `200` |
| `http://rwdkn.localhost/explorer/ground-truth/` | SSO-gated; authenticated response `200` |
| Ground Truth API | Healthy; 8 candidates from run `20260715_135426` |
| `http://ckn.localhost/` | `200`; participant landing page and API status |
| `http://gkn.localhost/` | `200`; participant landing page and API status |
| `http://hphyskn.localhost/` | `200`; participant landing page and API status |
| Participant `/api/{health,manifest,capabilities}` | `200` for CKN, GKN, and HPHYSKN |

Ground Truth ownership is now corrected:

- `rwdkn-service` owns the Ground Truth API, UI, container, route, tests, seed, and persistent state volume.
- `rwdkn-data-pipeline` produces immutable candidate/audit data but no longer owns the deployable curation service.
- Mutable reviews and manual triples belong in the `rwdkn-ground-truth-data` volume, not Git.

## 2. Important cautions before tomorrow's work

1. Do not assume the July 4 Linode state in older runbooks is still current. Re-inventory the live provider, Jenkins, Cloudflare, DNS, and containers first.
2. Do not use `CLEAN=wipe`, `docker compose down -v`, or a Pulumi destroy until persistent Ground Truth and database data are backed up and the target is confirmed.
3. Keep code ownership separate from data exchange. The pipeline may publish a release/audit artifact; the service must consume it through an explicit deployment contract, not a sibling source-tree dependency.
4. `kg-docs/architecture/repository-split.md` is now stale: it still assigns Ground Truth to `rwdkn-data-pipeline`. Update it after the deployment model is finalized.
5. The new CKN/GKN/HPHYSKN landing pages work locally but require public redeployment before their public roots can be considered fixed.

## 3. Tomorrow's recommended order

### Phase A — Inventory before mutation

- [ ] Record current Git SHA, image tag, container status, networks, volumes, and configured hosts for every deployable.
- [ ] Record the actual Linode instance ID/IP, Pulumi stack state, Cloudflare tunnel state, DNS targets, and Jenkins job branch/SHA.
- [ ] Confirm whether the public stack is running, stopped, partially deployed, or billing unexpectedly.
- [ ] Capture a route matrix before changing anything.
- [ ] Back up or snapshot PostgreSQL, Neo4j, and `rwdkn-ground-truth-data` before a destructive clean deployment.

Suggested local inventory:

```bash
cd /home/chandrakola/development/NLM/kg
./nlmkn/scripts/run_local_stack.sh status
docker compose ls
docker network inspect nlmkn-edge
docker volume inspect rwdkn-ground-truth-data
```

### Phase B — Clean localhost reproduction

- [ ] Stop the stack with `./nlmkn/scripts/run_local_stack.sh down` without deleting persistent volumes.
- [ ] Check for orphan containers and duplicate Compose project names.
- [ ] Start only through `./nlmkn/scripts/run_local_stack.sh up`.
- [ ] Confirm that no manual container reload, rename, or ad hoc environment export is required.
- [ ] Repeat the full route matrix in a real browser, including console and network errors.
- [ ] Verify one SSO login covers Hub and RWDKN explorer routes.
- [ ] Verify participant roots, manifests, capabilities, entity queries, and Hub federation calls.
- [ ] Verify Ground Truth candidates load and a test review survives a non-destructive container recreation.

Local acceptance criteria:

```text
Hub:                SSO -> /hub/ -> 200
RWDKN health:       /api/health -> 200
RWDKN explorers:    authenticated UI and required API calls -> 200
Ground Truth:       UI -> 200, candidates non-empty, persistent state survives recreate
CKN/GKN/HPHYSKN:    / -> 200; /api/health, manifest, capabilities -> 200
Browser console:    no failed assets or cross-origin errors
Clean startup:      no manual fixes after run_local_stack.sh up
```

### Phase C — Twelve-Factor configuration-independence audit

Treat `architecture/twelve-factor-compliance.md` as the starting point, not proof of completion. Audit all twelve factors, with emphasis on Factor III.

| Factor | Tomorrow's evidence |
|---|---|
| I. Codebase | One repository per deployable boundary; no runtime import from sibling repositories |
| II. Dependencies | Python/Node/image dependencies declared and reproducible from a clean checkout |
| III. Config | Hosts, origins, credentials, paths, ports, and feature flags supplied through environment/config profiles |
| IV. Backing services | PostgreSQL, Neo4j, Redis, NameRes, and Ground Truth data addressed as replaceable resources |
| V. Build/release/run | Immutable image build, explicit release config, separate runtime start; no source bind mounts in production |
| VI. Processes | App processes stateless; mutable Ground Truth and database state only in persistent backing stores |
| VII. Port binding | Each service binds its configured internal port and is exposed only by ingress |
| VIII. Concurrency | Worker/process scaling controlled through configuration, not source edits |
| IX. Disposability | Health checks, graceful restart, idempotent startup, bounded shutdown/startup time |
| X. Dev/prod parity | Same images and route contracts locally and on Linode; only environment profiles differ |
| XI. Logs | Application/access/error logs go to stdout/stderr or an external collector, not local source files |
| XII. Admin processes | Migration, import, load, backup, and smoke operations run as explicit one-off jobs |

High-priority configuration checks:

- [ ] Eliminate production source-code bind mounts and directory escapes.
- [ ] Replace the localhost Ground Truth sibling bind default (`../../rwdkn-data-pipeline/...`) with an explicit audit-artifact import or named data-volume contract.
- [ ] Review the production Insight Studio FAERS mount; it must consume a release artifact, not pipeline source.
- [ ] Require an explicit immutable `IMAGE_TAG` in public deployment instead of silently falling back to `latest`.
- [ ] Confirm every public hostname, CORS origin, redirect, cookie domain, and Neo4j advertised address comes from environment configuration.
- [ ] Confirm local defaults are safe only for local use and cannot leak into Linode deployment.
- [ ] Confirm secrets come from Infisical/runtime environment and never appear in Git, rendered logs, Compose output, or Jenkins console output.
- [ ] Confirm deployment configuration can change from localhost to Linode without modifying Python, JavaScript, Java, HTML, or Nginx application logic.

Useful audit searches:

```bash
rg -n 'localhost|kolac\.us|/home/|/opt/|\.\./\.\./' \
  nlmkn rwdkn-service ckn gkn hphyskn shared-infra

rg -n 'latest|container_name|ports:|volumes:|env_file:|environment:' \
  '*/deploy' shared-infra/ingress
```

Classify every match. A value in an environment-specific Compose profile may be valid; the same value embedded in application code or a production-generic template is a failure.

### Phase D — Public Linode deployment review

- [ ] Compare deployed Git SHAs and image digests to `origin/main` for all repositories.
- [ ] Confirm Jenkins builds and pushes the new `real-kg-ground-truth` image.
- [ ] Confirm the public RWDKN Compose stack includes Ground Truth, its persistent volume, and an immutable candidate-data source.
- [ ] Redeploy CKN/GKN/HPHYSKN so their public roots serve the new landing pages.
- [ ] Verify Cloudflare tunnel → Traefik → participant/service routing, including forwarded scheme and relative redirects.
- [ ] Verify SSO cookies have the intended public domain, `Secure`, `HttpOnly`, and `SameSite` attributes.
- [ ] Keep participant `/api/*` policy deliberately public or protected according to the federation contract; do not expose curation writes unintentionally.
- [ ] Verify the Hub registry uses the actual public `base_url`, `viewer_url`, `api_base_url`, and health URLs.
- [ ] Run authenticated browser checks for Hub and every explorer, not only curl checks.
- [ ] Run the federation smoke gate and save a dated report.

Public acceptance criteria:

```text
Deployed SHA/image digest == intended release
All public routes use HTTPS and contain no localhost or :8080 redirects
One SSO session works across intended UI subdomains
Participant roots and APIs return expected content
RWDKN query, provenance, Insight, and Ground Truth flows work
Persistent state survives container replacement
No source-tree or sibling-repository dependency exists on the Linode host
Rollback and backup procedures are documented and tested
```

## 4. Expected deliverables from tomorrow's review

- [ ] A dated localhost verification report with route, browser, and persistence evidence.
- [ ] A dated Linode/public-cloud verification report with deployed SHAs and image digests.
- [ ] A Twelve-Factor compliance matrix marked `PASS`, `PARTIAL`, or `FAIL`, with file/line evidence.
- [ ] Small scoped fixes for confirmed violations, verified locally before public rollout.
- [ ] Updated `architecture/repository-split.md` reflecting service ownership of Ground Truth.
- [ ] Updated operational runbook that no longer references the pre-split `real-kg` layout.
- [ ] A rollback/backup note for PostgreSQL, Neo4j, Ground Truth state, and the Linode/Pulumi stack.

## 5. Definition of done

Tomorrow's work is complete when a clean checkout can deploy the same application code to localhost and Linode using different configuration profiles, all route and browser checks pass, mutable state is externalized, production uses immutable artifacts, and no application-code edit is required to change deployment environments.
