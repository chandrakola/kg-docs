---
type: spec
title: "RWDKN Repository Split Design Spec"
description: "Specification and partition layout details for splitting the monolithic rwdkn codebase into separate pipeline and service repositories."
tags: [architecture, rwdkn, split, refactoring]
version: 1.0.0
created: 2026-07-16
updated: 2026-07-16
links:
  - "../README.md"
  - "../../rwdkn-data-pipeline/README.md"
  - "../../rwdkn-service/README.md"
---

# RWDKN Repository Split Design Spec

This document records the architectural split of the monolithic `rwdkn` (Real-World Data Knowledge Network) codebase into two independent, decoupled sibling repositories:

1. **`rwdkn-data-pipeline`**: Handles data extraction, LLM assertions, concept normalization, KGX compilation, QA, and local curation.
2. **`rwdkn-service`**: Handles serving APIs (`query-app`, `sentence-api`, `invite-gate`), Nginx edge routing, SSO integrations, and deployment.

---

## 1. Core Decoupling Strategy

To isolate the repositories and enable independent CI/CD pipelines, several coupling points were resolved:

### A. Environment Configuration (`env.py` helper)
* **The Coupling:** Both data processing scripts and APIs imported environment configuration helpers from a shared module.
* **Solution:** Replicated `env.py` inside both repositories (e.g., `pipeline/shared/env.py` and `services/shared/env.py`) preserving identical interfaces while severing cross-repository importing.

### B. Ground-Truth Curation Tooling
* **The Coupling:** Curation scripts (`ground_truth_api.py`) and Web UI files lived inline with serving code.
* **Solution:** Relocated the entire curation stack to `rwdkn-data-pipeline` under `tools/curation/`. The curation data (`reviews.jsonl`, `manual_triples.jsonl`, `state.json`) is managed in a persistent volume `rwdkn-ground-truth-data` with a default bootstrap seed file committed at `tools/curation/seed/reviews.seed.jsonl`.

### C. docker-compose Separation
* **Before:** A single monolithic `docker-compose.apps.yml` defined serving and curation services.
* **After:** 
  * `rwdkn-service` contains the production/dev serving stack (`query-app`, `sentence-api`, `invite-gate`, and edge proxy configurations).
  * `rwdkn-data-pipeline` uses a dedicated `tools/curation/docker-compose.curation.yml` to spin up ground-truth-api and its persistent volumes.

---

## 2. Sibling Mounts & File System Separation

All sibling repository bind mounts and directory-escaping dependencies have been retired to support building and running the code on isolated machines:

1. **Production `insight-studio` FAERS CSV Mount:**
   * **Old:** Mounted `../../subprojects/faers/faers_glp1_four_ingredients_adverse_event_report_counts_with_ids.csv` directly from pipeline source trees.
   * **New:** Packaged inside the pipeline's release bundle, mounted from the extracted host release folder: `/opt/real-kg/release/faers_counts.csv`.
2. **`query-app` KGX & Manifest Mounts:**
   * **Old:** Mounted `subprojects/biolink-bridge/application/kg-data/kgx` and `application/pipeline_manifest.json` from pipeline paths.
   * **New:** Binds directly to the extracted host release bundle directory `/opt/real-kg/release/kgx` and `/opt/real-kg/release/pipeline_manifest.json`.
3. **CI Escape Linting:**
   * A Python helper script (`python -m lint.check_escapes`) was added to the lint stages of both build pipelines to fail builds if any directory escapes (e.g. `../subprojects`) are introduced.

---

## 3. Inter-Repository Contract & Release Bundle

Communication between the data pipeline and the API services is now governed by an immutable **Release Bundle** output by the pipeline:

* **Graph Data:** `nodes.jsonl` and `edges.jsonl` (KGX format).
* **Evidence Database:** `evidence.sql` (Postgres dump of sentences, triples, and provenance).
* **KG Metadata & Schema Attestation:**
  * `pipeline_manifest.json` containing run metadata, timestamps, and git commits.
  * Pinned Biolink Model schema versions and terminate metadata (UMLS, RxNorm, etc.).
* **Attestation Signature:** SHA-256 digests of all bundle files, signed by the release pipeline to ensure authenticity.
