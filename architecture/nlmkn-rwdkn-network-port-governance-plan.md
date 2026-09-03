# NLM-KN / RWDKN Network and Port Governance Plan

**Status:** Approved after independent third-party review  
**Prepared:** 2026-09-01  
**Scope:** Local development, CI/CD, HAProxy ingress host, Cloudflare Tunnel, and NLM-KN/RWDKN application services  
**Change policy:** This document is the approved governance plan for IP, port, and network management.

## 1. Purpose

Establish a single, reviewable source of truth for IP addresses, host ports,
container ports, public hostnames, service ownership, and CI isolation. The goal
is to prevent port conflicts and accidental interference between shared
infrastructure, local development, CI/CD, and public ingress.

## 2. Current evidence snapshot

The following observations were made read-only on 2026-09-01:

- The local host runs shared infrastructure including the unified dashboard on
  `3000`, MLflow on `5000`, Neo4j on `7474/7687`, Fuseki on `3030`, PostgreSQL
  on `5432`, Jenkins on `8080`, and Dependency-Track on `8091`.
- NLM-KN/RWDKN application containers were not running at the time of review.
- The local Compose design assigns RWDKN service ports internally: query app
  `8050`, sentence API `8087`, Ground Truth `8091`, Insight Studio `3001`,
  edge `8080`, and NLM-KN auth `8099`.
- The local REAL-KG application Compose file publishes its edge through the
  configurable `REAL_KG_EDGE_PORT`, defaulting to `8095`.
- The `haproxy` host is `192.168.1.103`; HAProxy and cloudflared were active.
- HAProxy owns public listeners on `80` and `443`, and also listens on `6433`
  and `8006`. Pi-hole owns `53`, `8080`, and `8443`.
- cloudflared runs as systemd service from `/usr/bin/cloudflared` using
  `/etc/cloudflared/config.yml`; its local control listener was
  `127.0.0.1:20241`.
- HAProxy configuration validation succeeded with
  `haproxy -c -f /etc/haproxy/haproxy.cfg`.
- HAProxy has backends/routes including `real_kg_edge`, MLflow, Jenkins,
  dashboard, DailyMed, and Airflow.

Evidence must be refreshed before implementation because listeners, containers,
DNS, and tunnel routes are runtime state.

## 3. Target ownership model

| Layer | Owner | Rule |
|---|---|---|
| Public HTTP/HTTPS | HAProxy on `haproxy` | Only HAProxy binds public `80/443`. |
| Public tunnel | cloudflared | One explicitly owned tunnel configuration; no competing manager. |
| Docker ingress | Traefik/edge proxy | Routes by hostname to internal services. |
| Shared data services | Shared Docker Compose | Stable, documented ports; local-only bindings where possible. |
| NLM-KN/RWDKN apps | Application Compose | Internal Docker ports; no direct host publication unless explicitly approved. |
| Local development | Developer Compose/processes | Loopback-only and allocated development ports. |
| CI/CD | CI-specific Compose project | Isolated project, network, volumes, and ephemeral ports. |

## 4. Port allocation policy

### Stable local infrastructure

| Port | Current owner | Exposure policy |
|---:|---|---|
| 3000 | Unified dashboard | `127.0.0.1` |
| 3030 | Fuseki | `127.0.0.1` |
| 5000 | MLflow | `127.0.0.1` or protected proxy route |
| 5432 | Shared PostgreSQL | `127.0.0.1` |
| 7474/7687 | Shared Neo4j HTTP/Bolt | `127.0.0.1` |
| 8080 | Jenkins (local host) / internal proxy ports | Never reuse as a new host binding |
| 8084 | Infisical (local host) | Host 8084 maps to container 8080 |
| 8091 | Dependency-Track (local host) / Ground Truth internal port | Keep Ground Truth internal |
| 8095 | Local REAL-KG edge | Configurable, loopback-only |

### Internal application ports

Keep query app `8050`, sentence API `8087`, Ground Truth `8091`, Insight Studio
`3001`, edge `8080`, and auth `8099` as container ports. Reach them through
`nlmkn.localhost`, `rwdkn.localhost`, and `browser.localhost` via the local
reverse proxy rather than publishing them directly.

### CI/E2E ports

Use no host publication by default. When browser tests require host access,
allocate from `13000-13999` or ask Docker for an ephemeral port. Never bind CI
to shared ports `3000`, `5000`, `8080`, `8084`, `8091`, `7474`, `7687`, or `5432`.

## 5. Verification work packages

### WP-1: Inventory and ownership

Build a machine-readable registry containing service, owner, host/IP, listen
address, host port, container port, hostname, environment, data sensitivity,
and restart owner. Reconcile Docker, systemd, HAProxy, DNS, and cloudflared.

### WP-2: Local Compose isolation

Confirm that application Compose files use Docker service names for
container-to-container traffic, avoid unnecessary host bindings, and expose
only the approved local edge port.

### WP-3: HAProxy and cloudflared boundary

Verify that Cloudflare DNS points to the intended tunnel, cloudflared routes to
the intended HAProxy/edge origin, HAProxy backends resolve to the intended
service, and no redirect leaks internal ports such as `:8080`.

### WP-4: CI isolation

Require a unique Compose project name such as `ci-${CI_JOB_ID}`, a private
network, isolated data volumes when needed, no shared-service restarts, and
cleanup on both success and failure.

### WP-5: Health and drift checks

Automate checks for duplicate listeners, unexpected public binds, unhealthy
backends, stale Compose projects, tunnel failures, DNS drift, certificate
expiry, and failed CI cleanup.

## 6. Acceptance criteria

- No two approved services claim the same host IP/port pair.
- HAProxy remains the sole public `80/443` owner on the ingress host.
- cloudflared has one documented owner and one reviewed ingress configuration.
- NLM-KN/RWDKN backends are reachable through approved hostnames without direct
  host-port collisions.
- CI can run while shared local infrastructure remains untouched.
- A failed CI run cannot leave its ports, network, or volumes blocking the next run.
- The registry, Compose configuration, HAProxy route map, and tunnel routes agree.
- Independent reviewers can reproduce every claim from timestamped commands and
  configuration references.

## 7. Third-party review protocol

Each reviewer should record:

1. Review date, host(s), branch/commit, and tool used.
2. Commands or evidence sources inspected.
3. Findings classified as blocker, high, medium, low, or informational.
4. Exact proposed edits to this plan or the owning configuration.
5. Verification results for every changed claim.

Reviewers should not stop/restart services, rotate credentials, change DNS,
modify tunnel routes, or alter firewall/HAProxy configuration as part of review.

## 8. Review and update log

| Round | Date | Reviewer | Scope | Result | Follow-up |
|---|---|---|---|---|---|
| 0 | 2026-09-01 | Codex | Local ports, Compose definitions, HAProxy/cloudflared live state | Draft issued | Independent review required |
| 1 | 2026-09-01 | Google Gemini (Architect) | Infisical, Jenkins, `manage-lab.sh`, Docker state, federation stack | **APPROVED** | Proceeding with WP-1 implementation |



## 9. Implementation record

- 2026-09-01: Added `docs/specs/port_registry.json` as the machine-readable port allocation source.
- 2026-09-01: Added `scripts/validate_port_governance.sh`; the NLM-KN local stack runs it before startup.
- 2026-09-01: Bound the local REAL-KG edge to `127.0.0.1` by default and moved the assertion-viewer convenience port from conflicted `8082` to loopback-only `18082`.
- HAProxy and cloudflared remain unchanged; their live routing is a separate change-controlled ingress rollout.
- 2026-09-03: Added `platform-services/config/network-registry.json` as the canonical, full WP-1 registry (44 services, all 4 hosts, ownership + data-sensitivity metadata) alongside `platform-services/config/network-registry.schema.json` and `platform-services/tests/test_network_registry.py`.
- 2026-09-03: `docs/specs/port_registry.json` (as it was then) became a **generated** file, derived from the canonical registry by `platform-services/scripts/generate_port_registry.py`. Do not hand-edit the derived file; edit `network-registry.json` and regenerate.
- 2026-09-03: Fixed a collision-detection gap in `validate_port_governance.sh` — the uniqueness check keyed only on `bind:port`, which produced false positives once the registry spans multiple physical hosts (e.g. two different hosts both binding `0.0.0.0:8080`). The check now keys on `host:bind:port`.
- 2026-09-03: **Moved this entire governance surface out of `rwdkn-service`.** Port/IP governance is shared infrastructure, not application business logic, and `nlmkn/scripts/run_local_stack.sh` calling into a sibling *application* repo (`rwdkn-service`) for it was a dependency inversion. New locations:
  - This plan document: `rwdkn-service/docs/specs/` → `kg-docs/architecture/` (shared docs referenced by the whole federation, not owned by one app).
  - `scripts/validate_port_governance.sh`: `rwdkn-service/scripts/` → `platform-services/scripts/`.
  - The derived registry: `rwdkn-service/docs/specs/port_registry.json` → `platform-services/config/port_registry.json`. It is now generated *within* platform-services (no cross-repo write).
  - `nlmkn/scripts/run_local_stack.sh` now calls `platform-services/scripts/validate_port_governance.sh` directly. `rwdkn-service`, `nlmkn`, and every other federation participant depend on `platform-services` symmetrically for this; none depend on each other for it.
  - `platform-services/tests/test_port_registry_sync.py` and the `Validate Network Registry` Jenkins stage (in `platform-services`) fail the build if the canonical registry and the derived file drift apart.
