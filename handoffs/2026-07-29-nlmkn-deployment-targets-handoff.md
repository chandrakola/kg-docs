---
type: handoff
title: "NLM-KN Deployment Targets Handoff — 2026-07-29"
description: "Next-session checklist for validating the config-driven Linode deployment and defining PVE2."
tags: [handoff, nlmkn, deployment, localhost, linode, pve2, jenkins, traefik, infisical]
version: 1.0.0
created: 2026-07-28
updated: 2026-07-28
links:
  - "../../nlmkn/docs/deployment-configuration.md"
  - "../../nlmkn/deploy/targets/localhost.json"
  - "../../nlmkn/deploy/targets/linode.json"
  - "../../nlmkn/Jenkinsfile"
---

# NLM-KN Deployment Targets Handoff — 2026-07-29

**Prepared:** 2026-07-28 01:17 EDT  
**For:** 2026-07-29 work session  
**Repository:** `/home/chandrakola/development/NLM/kg/nlmkn`  
**Branch:** `main`  
**Published commit:** `5b4979d6cdc5fd4a9915200dc0db0e79b9f7bd03`  
**Primary goal:** Prove the generic deployment descriptor on Linode, then define and deploy the PVE2 target without changing Jenkins or application code.

## Tonight's completed state

- `5b4979d feat(deploy): add config-driven deployment targets` is committed and pushed to `origin/main`.
- `d13f123 fix(auth): route SSO login for all KN hosts` was also pushed.
- `deploy/targets/localhost.json` and `deploy/targets/linode.json` are the checked-in non-secret deployment records.
- `scripts/deployment_config.py` validates a deployment ID and derives hostnames, URLs, protocols, cookie policy, and Infisical references.
- Jenkins now accepts one `DEPLOYMENT_ID` instead of separate host and URL parameters.
- The localhost launcher uses the same renderer instead of maintaining a second hard-coded URL map.
- All participant hostnames and `SSO_COOKIE_SECURE` now reach the Hub/auth Compose configuration.
- Secrets remain in Infisical and `/run/nlmkn/sso.env`; they were not moved into deployment descriptors.

## Verification completed tonight

- [x] Five deployment configuration unit tests passed.
- [x] Localhost registry rendered with the expected RWDKN, CKN, GKN, and HPHYSKN URLs.
- [x] `scripts/run_local_stack.sh` passed `bash -n`.
- [x] Localhost and Linode Compose configurations rendered successfully.
- [x] `git diff --check` passed before commit.
- [x] `main` pushed successfully to `origin/main`.

## Important working-tree note

`nlmkn/docs/reports/local-smoke.md` remains modified locally and was deliberately excluded from the commit. It is generated evidence from the latest local smoke run. Review its changed timestamp/counts and either commit it as a separate evidence update or discard it deliberately; do not accidentally include it in an unrelated deployment commit.

## Tomorrow's TODO

### P1 — Validate Linode through Jenkins

- [ ] Confirm Jenkins has fetched commit `5b4979d` from `origin/main`.
- [ ] Run the NLM-KN job with `DEPLOYMENT_ID=linode` and the intended `SCOPE`/`CLEAN` values.
- [ ] Confirm the load-deployment stage reports `real-kg-demo`, `https`, and `nlmkn.kolac.us`.
- [ ] Confirm the rendered remote `deploy/deployment.env` contains no secret values.
- [ ] Confirm `/run/nlmkn/sso.env` remains mode `0640` and contains only runtime SSO credentials.
- [ ] Verify Traefik registered the Hub, health, and `/__auth` routers for every participant hostname.
- [ ] Verify the Hub homepage, login form, unauthenticated redirect, authenticated registry, and all participant links.
- [ ] Specifically retest the RWDKN explorer redirect that originally failed.
- [ ] Run/save the federation smoke evidence and record deployed image tag/SHA.

### P2 — Define PVE2

- [ ] Decide whether PVE2 is internal-only, public, or both.
- [ ] Record the SSH/Tailscale hostname used by Jenkins.
- [ ] Choose the DNS suffix and six hostnames: NLMKN, RWDKN, Browser, CKN, GKN, HPHYSKN.
- [ ] Decide HTTP versus HTTPS and the shared SSO cookie domain.
- [ ] Create `nlmkn/deploy/targets/pve2.json` by copying `linode.json` and setting `metadata.name` to `pve2`.
- [ ] Create or select the PVE2 Infisical environment and confirm Jenkins can reach it.
- [ ] Provision DNS, Traefik, Docker, and the external `nlmkn-edge` network on PVE2.
- [ ] Validate with `python3 scripts/deployment_config.py pve2 --format json` and Compose config rendering.
- [ ] Run Jenkins with `DEPLOYMENT_ID=pve2`; no Jenkinsfile case or application-code edit should be needed.

### P3 — Close evidence and documentation

- [ ] Review the local `docs/reports/local-smoke.md` change separately.
- [ ] Record Linode and PVE2 route matrices, image tags, Git SHAs, and smoke results.
- [ ] Update this handoff or create a dated completion report after both deployments are verified.

## Acceptance criteria

Tomorrow's deployment work is complete when:

1. The current Linode deployment succeeds using only `DEPLOYMENT_ID=linode`.
2. Homepage, SSO redirect/login, shared cookie behavior, participant routes, and federation smoke tests pass.
3. A valid `pve2.json` can drive the same Jenkinsfile without code changes.
4. PVE2 DNS/Traefik/runtime configuration matches its descriptor and its smoke tests pass.
5. No secret value appears in Git, Jenkins logs, or the checked-in descriptors.
