# Session Handoff — 2026-09-18

Written for resuming this work after a reboot or with a different AI tool.
Scope: `~/development/NLM/kg/` federated NLM-KN workspace + `~/development/workspaces/ai_lab/`
+ Jenkins/Infisical lab infra. Read `~/development/.claude/CLAUDE.md` and
`~/development/NLM/kg/CLAUDE.md` first — repo layout, `real-kg` naming landmines, lab service
inventory. Also read yesterday's handoff, `NLM/kg/SESSION_HANDOFF_2026-09-17.md`, and the
incident-specific doc this session produced: `NLM/kg/shared-infra/docs/2026-09-18-federation-outage-handoff.md`.

## ⚠️ Unfinished — pick up here first

**The kolac.us federation is still down (530 on nlmkn/rwdkn/ckn/gkn/hphyskn/browser).** The
Cloudflare token permission gap that caused it is fixed and verified, but the fix hasn't been
applied yet:

1. Trigger `http://localhost:8080/job/shared-infra-deploy/` → "Build with Parameters" →
   **`ACTION=ingress`** (it defaults to `preview` if you don't change the dropdown — that's
   what happened on the first retry attempt, build #72, which only previewed and changed
   nothing). Other params: `PROVIDER=linode`, `FOUNDATION_ENVIRONMENT=demo`,
   `PULUMI_STACK=foundation`, `DEPLOY_HOST=real-kg-demo`, `PUBLIC_BASE_DOMAIN=kolac.us`.
2. Confirm all six hosts return non-530.
3. **Do not run `ACTION=up`** until the separate VM-replace-on-every-run bug (below) is
   fixed — it will force another full VM replace + reboot cycle for no reason.

## Repos/systems touched this session

| Repo/system | Path | Commit(s) |
|---|---|---|
| `ai_lab` | `~/development/workspaces/ai_lab/` | `e72cd7f` (reports.kolac.us shared SSO) |
| `rwdkn-service` | `NLM/kg/rwdkn-service/` | `8361181` (catalog index.html latest-run fix) |
| `shared-infra` | `NLM/kg/shared-infra/` | docs-only, `docs/2026-09-18-federation-outage-handoff.md` (uncommitted) |
| Jenkins (`localhost:8080`) | — | deleted 6 dead jobs (see below); no repo commit |
| Infisical (`localhost:8084`) | — | `CLOUDFLARE_API_TOKEN` permissions fixed in `shared-infra-zbvy`/`demo`; no projects deleted |
| Elasticsearch (`localhost:9200`) | — | de-aliased 2 stale physical indices (see below) |

`shared-infra`'s doc is **not yet committed** — worth committing before it's forgotten.

## What we did, chronologically

1. **`reports.kolac.us` → shared SSO with the hub.** User wanted the same login as
   `nlmkn.kolac.us/hub/`. Discovered `reports.kolac.us` runs through a *completely different*
   ingress than the NLM-KN federation: a dedicated `haproxy` box (LAN `192.168.1.103`, SSH alias
   `haproxy`, **not** reachable via Tailscale — that tailnet node is stale/467+ days offline) that
   was previously gated by Cloudflare Access, not Traefik/nlmkn-sso. Deployed a second
   `nlmkn-auth` instance (`nlmkn-auth-reports`) sharing the hub's `SSO_PASSWORD`, wired via nginx
   `auth_request` (had to work around `/verify`'s Traefik-ForwardAuth-style 302-on-failure, which
   nginx treats as an upstream error, not a denial — fixed with `proxy_intercept_errors` +
   `error_page 302 = @denied`). User removed the Cloudflare Access application manually.
   Verified end-to-end via a real login round-trip. Full reusable recipe in memory
   `project_kolac_us_sso_pattern`.

2. **Fixed a stale report catalog card.** `rwdkn.kolac.us/explorer/catalog/20260916_182943.html`
   404'd — turned out to be a **deploy gap**, not a bug: the file was correctly generated,
   committed, and pushed, just never deployed to `real-kg-demo`. Along the way, found the "Latest
   Quality Dashboard" card in `docker/explorer/catalog/index.html` was hardcoded/stale (pointed
   at the 09-09 run); the "Pipeline Run Reports" grid below it is *not* stale — it's populated
   live from `/explorer/api/stats`. Fixed the stale card only (`rwdkn-service@8361181`). **The
   underlying deploy is still pending** — same `shared-infra-deploy` ingress fix above will need
   a `rwdkn/pipeline` run afterward to actually publish it.

3. **Diagnosed the federation outage** (`nlmkn-coordinator` #104, then #105 ~70 min later,
   triggered from the Jenkins UI by the user both times — not a rogue webhook, confirmed no
   `<triggers/>` configured on the job). Two distinct root causes, both in
   `shared-infra/docs/2026-09-18-federation-outage-handoff.md`:
   - **Cloudflare token missing DNS permission** — token id `4f356919f196c37b1483c3b3082eeef6`
     was active and could list zones, but had zero DNS read/write permission on `kolac.us`.
     **Fixed and verified** (live round-trip DNS write test: created + deleted a throwaway TXT
     record successfully).
   - **Every `ACTION=up` replaces the entire Linode VM**, not just DNS —
     `LinodeDemoEnvironment.java`'s `.replaceOnChanges("environment")` never stabilizes
     (`[diff: +addPreviousOutputInEnv~environment]` on every single run). VM does get deleted
     before replace (no orphan billing), but it's a full downtime cycle every trigger.
     **Not fixed** — needs its own investigation into what's making `environment` non-deterministic.

4. **Restarted the local dev stack** after the user ran `manage-lab.sh stop` /
   `manage-apps.sh stop` (confirmed via journal this was unrelated to the outage — ran afterward,
   local-only). `manage-lab.sh start full` succeeded (minio transient-failed then came up fine
   standalone). **Found `manage-apps.sh start all` is broken**: it runs `docker compose up -d`
   from each app's own `deploy/` directory with no `-p` flag, and `nlmkn/deploy/`,
   `rwdkn-service/deploy/`, `ckn/deploy/`, `gkn/deploy/`, `hphyskn/deploy/` are all directories
   literally named `deploy` — Compose's default project-name-from-basename collapses them into
   one project called `deploy`, so each app start tears down the previous one. Worked around by
   using `NLM/kg/nlmkn/scripts/run_local_stack.sh up` instead (correct distinct project names
   already). **`manage-apps.sh` itself is not yet fixed.**

5. **Jenkins cleanup** — deleted 6 dead/stale jobs (all confirmed via build history + SCM source
   inspection, not guessed): `real-kg`, `real-kg-demo` (both sourced from the retired
   pre-rename `real-kg.git` repo — `real-kg-demo` had zero builds ever), `HelloWorld`,
   `test-job-pipeline`, `dailymed-spl-kg` (last run 2026-05-25), `VerificationPipeline` (last run
   2025-12-28). 15 jobs remain, all confirmed active within the last 2 months except the two
   nested-folder ones (`rwdkn/pipeline`, `dailymed-search-fe/dailymed-fe-deploy`) which are fine.

6. **Fixed `dailymed-search-gpu-cuda-xpu`'s failing smoke test.** Root cause: both
   `fda_spl_chunks_v3` and `fda_spl_sections_v3` Elasticsearch aliases pointed at **two** physical
   dated indices simultaneously (complete `20260814` release + incomplete/stuck `20260824`
   release, 5.16M vs 2.37M docs — matches the repeated "orphaned tasks" Airflow scheduler
   messages, i.e. that re-embedding campaign never finished). The app's readiness contract
   requires exactly one physical index per alias. The pipeline's own alias-cutover step is
   deliberately unautomated (unit-tested to always return `alias_cutover_performed: False` —
   manual promotion by design, not a bug). Fixed by removing the incomplete index from both
   aliases (reversible, no data deleted); `/health` now reports `embedding.ready: true`.
   **Job itself hasn't been re-triggered to confirm green** — Jenkins blocks anonymous build
   triggers (403, confirmed it's Jenkins auth, not the harness) so this needs a manual
   "Build Now" click.

7. **Infisical inventory & cleanup investigation.** Full org has 1 org, 8 projects. Two land-mine
   findings:
   - **Two projects both display as "real-kg"** with different slugs — `real-kg-ww-mv` (live,
     25-31 secrets/env, hardcoded fallback workspace in `foundation_up.sh`) vs `real-kg-x0-th`
     (a **KMS**-type project, 0 keys, 0 activity ever — genuinely dead by every signal checked:
     code refs, audit log, integrations, identity memberships, not used as any other project's
     KMS backend).
   - **`infisical init`'s interactive picker resolved "shared-infra-zbvy" to the wrong project**
     once (landed on `rwdkn` instead) — this produced a false "token is missing everywhere"
     conclusion that cost real time before being caught and corrected. Always verify the resolved
     `workspaceId` against the known project-id table in the incident handoff doc.
   - Deep-analyzed `MLflow v2`, standalone `pve2`, and `real-kg-x0-th` as deletion candidates
     (zero secrets/keys, zero integrations, zero audit log entries, zero code references, all
     independently confirmed). **User declined to delete anything** — left all as-is. Real
     duplication also found (`CLOUDFLARE_API_TOKEN` copy-pasted into ~9 places,
     `DOCKER_HUB_USER`/`PAT` into ~18) — flagged as a "use Infisical secret references instead of
     copies" follow-up, not acted on.

8. **Wrote the incident handoff doc** (`shared-infra/docs/2026-09-18-federation-outage-handoff.md`,
   uncommitted) and a memory entry (`project_federation_outage_2026-09-18`) covering both root
   causes in full detail — read that doc before re-investigating any of this from scratch.

## Current live status (as of session end)

| Target | Status | Notes |
|---|---|---|
| `nlmkn.kolac.us` / `rwdkn.kolac.us` / `ckn` / `gkn` / `hphyskn` / `browser`.kolac.us | ❌ 530 | DNS never got cut over — needs the `ACTION=ingress` run above |
| `reports.kolac.us` | ✅ Healthy | New shared-SSO login, verified live |
| Local federation (`*.localhost` via Traefik) | ✅ Healthy | `nlmkn-hub`, `rwdkn-local`, `kn-ckn/gkn/hphyskn` all under correct distinct Compose projects |
| Jenkins (`localhost:8080`) | ✅ Healthy | 15 jobs, all confirmed active |
| `dailymed-search-api-i9` (`localhost:8001`) | ✅ Healthy | `embedding.ready: true`; Jenkins job not yet re-triggered to confirm green |
| Cloudflare token (`shared-infra-zbvy`/`demo`) | ✅ Fixed & verified | DNS read+write round-trip tested live |

## Credentials / access notes for whoever picks this up

- **SSH to `real-kg-demo` is blocked by Tailscale's tailnet ACL policy** (not a Claude Code
  permission — "tailnet policy does not permit you to SSH as user chandrakola"). This needs
  fixing in the Tailscale admin console, or do it from a session that already has access.
- **`haproxy` host**: SSH alias `haproxy` (LAN `192.168.1.103`, user `ubuntu`) — not on Tailscale,
  the tailnet node by that name is stale/unrelated. `/etc/haproxy/haproxy.cfg` is the single
  source of truth for which `*.kolac.us` host maps to which backend — check it directly rather
  than inferring from any docker-compose file, since most home-lab services aren't wired through
  Traefik/Cloudflare Tunnel at all.
- **Jenkins anonymous access**: reads and build-*triggering* work anonymously (crumb from
  `/crumbIssuer/api/json`, then POST with `Jenkins-Crumb` header — no cookies needed this
  session, unlike the 09-17 note). **Job deletion and Cloudflare/Infisical/Pulumi writes are
  blocked** — both by Jenkins' own auth (403 on `doDelete`) and by the Claude Code harness
  classifier for anything touching production DNS/secrets/deploys. Expect to hand those off to
  a human either way.
- **Infisical**: self-hosted at `localhost:8084`, one org ("Admin Org"), 8 projects — see the
  incident doc for the full project-id table. `infisical login` needs a real terminal (arrow-key
  picker doesn't work over the `!`-prefix non-interactive path) — ask the user to run it
  themselves. Once logged in, `infisical user get token` gives a session JWT usable for direct
  REST calls against `localhost:8084/api/...` for anything the CLI doesn't expose (e.g. cross-
  project listing, KMS, audit logs).

## Where to look first if resuming

1. Read `~/development/.claude/CLAUDE.md` + `~/development/NLM/kg/CLAUDE.md`, then this doc's
   "Unfinished" section at the top.
2. Check `shared-infra/docs/2026-09-18-federation-outage-handoff.md` for the full outage
   root-cause writeup before re-diagnosing anything Cloudflare/DNS/VM-related.
3. `curl -o /dev/null -w '%{http_code}\n' https://nlmkn.kolac.us/hub/` (and the other 5 hosts) to
   see if the `ACTION=ingress` fix ever got applied.
4. `docker compose ls` to confirm the local stack is still healthy and no project-name
   collisions crept back in.
