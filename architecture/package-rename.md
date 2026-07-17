---
type: spec
title: "Shared Infrastructure Package Namespace Refactoring"
description: "Details the package rename from us.kolac.rkg to us.kolac.kg in shared-infra, VM/DNS provisioning changes, and CI coordination pipeline enhancements."
tags: [shared-infra, refactoring, maven, java, pulumi, cloudflare]
version: 1.0.0
created: 2026-07-16
updated: 2026-07-16
links:
  - "../README.md"
  - "../../shared-infra/README.md"
---

# Shared Infrastructure Package Namespace Refactoring

This document records the namespace refactoring and pipeline optimizations applied to the `shared-infra` project to streamline deployment lifecycle management and standardize naming.

---

## 1. Package Renaming (`us.kolac.rkg` to `us.kolac.kg`)

To align with the unified `kg` (Knowledge Graph) workspace design and remove the legacy `r` prefix (formerly "Real-World"):

* **Java & Maven Refactoring:**
  * Renamed the primary package directory tree from `us/kolac/rkg` to `us/kolac/kg`.
  * Updated references across `EnvConfig.java`, `AccessGatewayProvisioner.java`, `CloudflareDemoProvisioner.java`, `DnsProvisioner.java`, `WafProvisioner.java`, and provider-specific provisioning structures (e.g. Linode, AWS, GCP, Azure).
* **Config Uniformity:**
  * Aligned system variables and configuration references to use the clean `.kg` suffix or namespace identifier.

---

## 2. Ingress & DNS Configuration Updates

The infrastructure orchestrator added support for new subdomain routes and external participant viewers:
* **DNS Ingress:** Added `browser.kolac.us` as a foundation participant host.
* **Redirection & Gateway Config:** Configured ingress proxies to route traffic correctly to dedicated ports on the foundation VMs without certificate mismatch errors.

---

## 3. CI Coordinator & Lifecycle Pipeline Upgrades

Significant improvements were made to how the coordination and deployment pipelines build and run tasks:

* **Parallelized Simulators (`nlmkn-coordinator`):**
  * Updated the coordinator to execute build and run sequences for simulated fixtures (like `ckn`, `gkn`, `hphyskn`) in parallel instead of sequentially. This reduced total integration runtimes significantly.
* **Jenkins coordinator Design (`shared-infra-deploy`):**
  * Split the Pulumi and Maven infrastructure orchestrations into independent steps.
  * Implemented a parameterized parallel fan-out structure to coordinate child jobs dynamically based on deployment flags.
  * Added SSH `ControlMaster` socket caching to share TCP connections between steps, preventing connection limits and latency overhead during provisioning.
