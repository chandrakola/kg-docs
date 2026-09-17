# Session Handoff — 2026-09-17

Written for resuming this work after a reboot or with a different AI tool.
Scope: `~/development/NLM/kg/` federated NLM-KN workspace (see `~/development/NLM/kg/CLAUDE.md`
and `~/development/.claude/CLAUDE.md` for project conventions — read those first, they explain
the repo layout, `real-kg` naming landmines, and lab service inventory).

## Repos touched this session (all pushed, nothing uncommitted left behind)

| Repo | Path | Latest relevant commit |
|---|---|---|
| `rwdkn-data-pipeline` | `NLM/kg/rwdkn-data-pipeline/` | `068dd8a` (S3 delta-sync fix, catalog update) |
| `rwdkn-service` | `NLM/kg/rwdkn-service/` | `2e90073` (status badge fix) — also `ba9dbae` (minio quay.io fix), `f6764ad` (catalog) |
| `shared-infra` | `NLM/kg/shared-infra/` | `8e688dd` (ARTIFACT_VERSION bump), `fe40997` (mise.toml) |
| `nlmkn` | `NLM/kg/nlmkn/` | `c37ed0e` (pve2 hostname fix) |
| `ai_lab` | `~/development/workspaces/ai_lab/` | `e724edb` (lab-reports mount fix) |

Run `git -C <repo> status` / `git log --oneline -5` in each to re-orient. `ai_lab` has 2 other
pre-existing modified files (`configs/jenkins/Dockerfile`, `storage/GPUBenchmarking.ipynb`) that
are **not mine** — leave them alone unless the user says otherwise.

## What we did, chronologically

1. **Fixed `shared/s3_utils.py`** (`rwdkn-data-pipeline`) — `upload_directory()` was re-uploading
   every file on every call regardless of whether it changed. Added ETag/MD5 delta-check,
   defaults to skip-unchanged now; `--full` CLI flag forces old behavior.
   ⚠️ **`scripts/s3_storage.py` (the older, separate S3 helper module) was NOT fixed** — it still
   does a full re-upload every time. Used by `run_all.sh`'s cache-mirror/run-snapshot uploads.

2. **Removed a redundant S3 sync** in `subprojects/biolink-bridge/run_reports.sh` — both callers
   (`run_all.sh`, root `run_reports.sh`) already re-sync the full reports tree right after it runs.

3. **Added a PMID filter** to the pubmed KGX explorer report
   (`rwdkn-data-pipeline/subprojects/biolink-bridge/generate_report.py`) — SME can paste
   10-20 PMIDs (comma/space/newline separated) to review one article's extracted rows at a time,
   grouped by PMID. Verified via headless browser testing (Playwright). Tests added in
   `tests/test_generate_report.py`.

4. **Fixed `localhost:8086` (local `lab-reports` nginx dashboard)** — was serving 13-day-stale
   content from an orphaned `/usr/kpcs/share_write/real-kg/reports/` path that nothing writes to
   anymore, AND 7 other mounts pointed at a deleted `rwdkn/` directory (repo was split into
   `rwdkn-service`+`rwdkn-data-pipeline` on 2026-08-07). Repointed all mounts in
   `~/development/workspaces/ai_lab/docker-compose.yml` directly at the live repo. Container
   recreated, verified serving current content, pushed as `ai_lab@e724edb`.

5. **Investigated Neo4j dailymed/pubmed duplication mystery** — traced to `--resume` in
   `pubmed_llm_spo.py` not recognizing legacy `<PMID>.json` files after a Sep-4 filename
   convention change to `<PMID>.abstract.json`, causing 316 PMIDs to be silently re-extracted
   (real API cost, ~48% of the pubmed corpus was duplicate). **Cleaned up the 316 stale files**
   (confirmed every one had a `.abstract.json` twin first). Re-ran the pipeline, confirmed zero
   new duplicates. No code fix was needed once the naming was understood correctly (files
   already existed in a self-healing config — see full reasoning in conversation if revisiting).

6. **Built and uploaded a fresh cloud-demo artifact** — `scripts/cloud_demo_build_artifact.sh
   --version 20260916` → `/usr/kpcs/share_write/real-kg/jenkins/cloud-demo-artifacts/20260916/`
   → uploaded to `s3://real-kg/runs/20260916/` via `scripts/s3_storage.py upload-dir`. Contains
   Postgres dump (36,012 triples / 25,550 sentences) + all 4 KGX sources.

7. **Bumped `ARTIFACT_VERSION` default** `20260707` → `20260916` in both
   `shared-infra/Jenkinsfile.coordinator` and `rwdkn-service/Jenkinsfile`.

8. **Deployed to `linode` (`nlmkn.kolac.us`) via `nlmkn-coordinator`** — hit and fixed 3 real bugs
   along the way (builds #90-92 failed, #93-94 succeeded):
   - `SCOPE=data` computes `IMAGE_TAG` from current git SHA even when that commit was never
     built/pushed → always fails after a config-only commit. Worked around by using
     `REFRESH_SCOPE=application`/`both` instead (doesn't fix the underlying script bug, just
     avoids it).
   - **MinIO discontinued Docker Hub distribution entirely** (`minio/minio` repo gone, confirmed
     via Docker Hub API). Fixed by repointing `deploy/rwdkn/docker-compose.rwdkn.yml`'s `minio`
     service image to `quay.io/minio/minio:RELEASE.2025-09-07T16-13-09Z` (`rwdkn-service@ba9dbae`).
   - `REFRESH_SCOPE=application` only redeploys the app, does NOT load data — needed a follow-up
     run with `REFRESH_SCOPE=both` to actually load `ARTIFACT_VERSION=20260916`.
   - Final state: **linode fully deployed, verified healthy**, data validated
     (25550/25550 sentences, 36012/36012 triples match manifest exactly).

9. **Fixed a status badge CSS bug** in `rwdkn-service/docker/explorer/sentences/index.html` —
   `LOADED`/`LOAD_FAILED` statuses (real values `shared/sentence_api.py` emits) had no matching
   CSS class, fell through to a base rule that forces white text with no background → invisible
   white-on-white pills. Added the 2 missing status styles across both CSS blocks in the file
   (`rwdkn-service@2e90073`). Deployed to linode via `REFRESH_SCOPE=application` (build #96,
   SUCCESS) since it's a static-asset-only change.

10. **Deployed to `pve2` (`rwdkn-pve2.kolac.us`)** — was failing because:
    - Tailscale was **logged out** on the VM (confirmed/fixed via QEMU guest agent through the
      Proxmox hypervisor, `ssh pve2` → `qm guest exec 104 -- tailscale status` /
      re-login — VM itself was healthy the whole time, just disconnected from Tailscale).
    - `nlmkn/deploy/targets/pve2.json` and 3 other files hardcoded the wrong Tailscale hostname
      `nlmkn-pve2-2` — the VM's real name (confirmed via `qm config 104` on the Proxmox host,
      VMID 104, `managed-by-pulumi;nlmkn` tag) is **`nlmkn-pve2`** (no `-2`). Fixed all 4
      references (`nlmkn@c37ed0e`), tests updated and passing.
    - Final state: **pve2 fully deployed, verified healthy**, same exact data validation as
      linode (25550/25550, 36012/36012).

11. **Checked `ckn`/`gkn`/`hphyskn`/hub** — all healthy. These are static-fixture participants
    (per CLAUDE.md), last deployed 2026-08-04, no redeploy needed since — their `/api/health`
    returns a hardcoded fixed `timestamp` confirming they're static content, not live services.
    Hub itself is SSO-gated; confirmed the SSO service's own health directly
    (`__auth/health` → ok), and confirmed the Hub journey indirectly via
    `federation-smoke-test`'s passing `LIVE_BROWSER=true` run.

12. **Investigated evidence/ground-truth file dependencies** (in response to "I thought
    everything was in Postgres") — found it is NOT:
    - `shared/ground_truth_api.py` (rwdkn-service) has **zero SQL** — the whole SME
      ground-truth review queue is JSONL/JSON files on disk
      (`triple_audit.jsonl`/`triple_rejected.jsonl`/`triple_skipped.jsonl` +
      glob'd PubMed article/assertion `.json` files).
    - `PUBMED_ARTICLE_DIR`/`PUBMED_ASSERTION_DIR` default to `rwdkn-data-pipeline` repo paths
      that **are not mounted into the deployed `rwdkn-ground-truth` container at all** — this
      silently produces an empty PubMed candidate list in production right now (confirmed by
      reading the compose file's actual volume list vs. what the code expects).
    - DailyMed SPL "evidence" HTML links (`generate_report.py`'s hardcoded
      `http://localhost:8082/{setid}.html`) rely on an `assertion_html_server` container that
      lives in `rwdkn-data-pipeline`'s separate `docker-compose.yml` — **confirmed via direct
      SSH to the pve2 host that this container is not running there at all.** Clicking a
      "View Local HTML" evidence link on the live site → 502.
    - Root cause for both gaps: leftover wiring assumptions from before `rwdkn` split into
      `rwdkn-service` + `rwdkn-data-pipeline` (2026-08-07), never reconnected.
    - **Not yet fixed** — this was where the session left off. See Outstanding Issues below.

## Current live status (as of session completion)

| Target | Status | Data version / Notes |
|---|---|---|
| `nlmkn.kolac.us` (linode) | ✅ Healthy | `ARTIFACT_VERSION=20260916`, Jenkins #98, #101 |
| `rwdkn-pve2.kolac.us` (pve2) | ✅ Healthy | `ARTIFACT_VERSION=20260916`, Jenkins #99 |
| `ckn.kolac.us` / `gkn.kolac.us` / `hphyskn.kolac.us` | ✅ Healthy | Static fixtures |
| Hub (`https://nlmkn.kolac.us/hub/`) | ✅ Healthy | Verified via Playwright live browser test |
| `localhost:8086` (local reports dashboard) | ✅ Fixed | Serving current live content |
| Ground-truth review API & candidates (Postgres) | ✅ Fixed & Active | 6,778 candidates live from PostgreSQL |
| DailyMed SPL evidence HTML viewer (production) | ✅ Fixed & Active | `assertion_html_server` deployed on Linode & PVE2 |

## Completed Implementation Tasks (September 17)

1. **PostgreSQL Ground-Truth Migration (Task 1 / P0)** — **COMPLETE**
   - Replaced JSONL filesystem dependence with direct PostgreSQL parameterized queries across `sentence_stage_events`, `triples`, and `triple_provenance`.
   - Added SME-owned tables `ground_truth_review` and `ground_truth_manual_triple` plus index `idx_stage_events_stage_created`.
   - Migrated on both Linode and PVE2. Verified via `/explorer/ground-truth/api/stats` and `/explorer/ground-truth/api/candidates`.

2. **DailyMed Evidence HTML Viewer (Task 2 / P1)** — **COMPLETE**
   - Added `assertion_html_server` (`nginx:alpine`) to `deploy/rwdkn/docker-compose.rwdkn.yml` mounting `/opt/nlmkn/rwdkn/artifacts/current/html:ro`.
   - Configured edge routes in `edge.rwdkn.nginx.conf.template` for `/assertions/` and `/explorer/assertions/`.
   - Parameterized report evidence URLs in `subprojects/biolink-bridge/generate_report.py` via `REAL_KG_EVIDENCE_BASE_URL`.
   - Synced 56 DailyMed HTML files to `/opt/nlmkn/rwdkn/artifacts/current/html` on both Linode and PVE2. Both return HTTP 200 for valid files and 404 for missing files.

3. **Jenkins `SCOPE=data` Refresh Logic (Task 3 / P1)** — **COMPLETE**
   - Updated `rwdkn-service/Jenkinsfile` to dynamically inspect the running container image (`docker inspect --format '{{.Config.Image}}' rwdkn-query-app`) when `SCOPE=data`.
   - Bypasses missing image failures on config-only commits.

4. **S3 Storage Delta Sync (Task 4 / P2)** — **COMPLETE**
   - Updated `rwdkn-data-pipeline/scripts/s3_storage.py` and `rwdkn-service/scripts/s3_storage.py` with MD5/ETag check (`_remote_matches`).
   - Added `--force`/`--full` and `--dry-run` flags, `download-dir` CLI command, and uploaded/skipped/failed summary reporting.
   - Added 9 unit tests in `tests/test_s3_storage.py` (all passing). Allowlisted in `check_manifest_coverage.py`.

5. **Session Handoff Hardening (Task 5 / P2)** — **COMPLETE**
   - Corrected all SSO health endpoints to `https://nlmkn.kolac.us/__auth/health` and `https://nlmkn-pve2.kolac.us/__auth/health`.
   - Corrected RWDKN API health checks to `https://rwdkn.kolac.us/api/health` and `https://rwdkn-pve2.kolac.us/api/health`.
   - Documented reproducible validation commands and commit SHAs.

## Credentials / access notes for whoever picks this up

- No login for `https://rwdkn.kolac.us/explorer/*` or `https://nlmkn.kolac.us/hub/` (SSO-gated) —
  only `/api/*` and `/__auth/health` are open without auth.
- No Docker Hub API auth for the private `chandrakola/rwdkn-query-app` repo (public API returns
  "object not found" for private repos without a token).
- Jenkins (`localhost:8080`) allows anonymous build triggering — no credentials needed, just a
  CSRF crumb (see any of tonight's `curl` calls in the conversation for the exact pattern:
  fetch `/crumbIssuer/api/json` with a cookie jar, then POST `buildWithParameters` with
  `Jenkins-Crumb` header + the same cookies).
- `ssh pve2` / `ssh nlmkn-pve2` both work from this machine (LAN + Tailscale respectively).
  `qm guest exec 104 -- <cmd>` runs commands inside the VM via the Proxmox hypervisor's QEMU
  guest agent, bypassing network entirely — useful if Tailscale ever drops again.
- Direct SSH to the deploy hosts themselves (`real-kg-demo`, etc.) for anything beyond
  read-only checks is gated by the harness's "Production Reads" classifier — expect prompts.

## How Jenkins deployment decisions work (reference — full explanation given earlier in session)

Three independent axes:
- **`DEPLOYMENT_ID`** (`linode`/`pve2`/`localhost`) → resolves `nlmkn/deploy/targets/<id>.json`
  → picks the target host/hostnames/foundation provider.
- **`DEPLOY_TYPE`** (`full`/`refresh`/`build-only`/`teardown`) → whether infrastructure
  (Pulumi via `shared-infra-deploy`) gets touched at all. `refresh` (used all session) never
  touches infra — assumes the VM already exists.
- **`SCOPE`/`REFRESH_SCOPE`** (`both`/`application`/`data`/`build-only`) → per-participant,
  independently of the above — controls whether `build and push` (code), `deploy` (redeploy
  containers), and/or `load data` (Postgres+Neo4j restore) actually run.
- `ARTIFACT_VERSION` only matters within the `load data` stage — picks which S3-hosted bundle
  (built by `cloud_demo_build_artifact.sh`) to restore.

## Where to look first if resuming


### 6. Full Code + Data Deployments to Linode and PVE2 (11:28 - 11:36 EDT)

1. **Artifact Bundle Rebuild & Sync (`20260916`)**:
   - Re-executed `scripts/cloud_demo_build_artifact.sh --version 20260916` to package Postgres dump, KGX exports, and all 56 DailyMed SPL HTML files into `/usr/kpcs/share_write/real-kg/jenkins/cloud-demo-artifacts/20260916/html`.
   - Uploaded bundle delta to MinIO S3 via `scripts/s3_storage.py upload-dir /usr/kpcs/share_write/real-kg/jenkins/cloud-demo-artifacts/20260916 runs/20260916` (62 files uploaded).
2. **Jenkins Coordinator Deployments (`REFRESH_SCOPE=both`)**:
   - **Linode (`real-kg-demo`)**: Triggered `nlmkn-coordinator` Build #102 (`DEPLOYMENT_ID=linode`, `REFRESH_SCOPE=both`, `ARTIFACT_VERSION=20260916`). Result: **SUCCESS** (180s duration; `rwdkn/pipeline` #30 **SUCCESS**).
   - **PVE2 (`nlmkn-pve2`)**: Triggered `nlmkn-coordinator` Build #103 (`DEPLOYMENT_ID=pve2`, `REFRESH_SCOPE=both`, `ARTIFACT_VERSION=20260916`). Result: **SUCCESS** (194s duration; `rwdkn/pipeline` #31 **SUCCESS**).
3. **Live Verification Matrix**:
   - Linode `https://rwdkn.kolac.us/api/health`: `status: ok`
   - Linode `https://rwdkn.kolac.us/assertions/0450d8a2-a88e-4849-9788-ed4f5246f223.html`: `HTTP/2 200`
   - Linode `ground-truth` PostgreSQL cache: `candidate_count: 6778`, `all_candidate_count: 33264`, `candidate_cache_ready: true`
   - PVE2 `https://rwdkn-pve2.kolac.us/api/health`: `status: ok`
   - PVE2 `https://rwdkn-pve2.kolac.us/assertions/0450d8a2-a88e-4849-9788-ed4f5246f223.html`: `HTTP/2 200`
   - PVE2 `ground-truth` PostgreSQL cache: `candidate_count: 6778`, `all_candidate_count: 33264`, `candidate_cache_ready: true`

---

## Next Session Checklist

1. Read `~/development/.claude/CLAUDE.md` and `~/development/NLM/kg/CLAUDE.md` for full repo/
   convention context (this doc assumes you've already read those).
2. `git -C <repo> log --oneline -10` in each repo above to confirm nothing has drifted since
   this doc was written.
3. Re-check live health before assuming anything is still true — things move fast in this lab:
   `curl https://rwdkn.kolac.us/api/health`, `curl https://rwdkn-pve2.kolac.us/api/health`,
   `curl http://localhost:8080/job/nlmkn-coordinator/lastBuild/api/json?tree=result`.

