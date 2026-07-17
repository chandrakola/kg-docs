---
type: index
title: "Knowledge Graph (KG) Documentation Hub"
description: "Centralized entrypoint and index for the NLM Knowledge Graph ecosystem documentation. Structured using the Open Knowledge Format (OKF)."
tags: [knowledge-graph, documentation, nlmkn, federation]
version: 1.0.0
created: 2026-07-16
updated: 2026-07-16
links:
  - "./architecture/repository-split.md"
  - "./architecture/package-rename.md"
  - "./architecture/twelve-factor-compliance.md"
  - "../ckn/README.md"
  - "../gkn/README.md"
  - "../hphyskn/README.md"
  - "../nlmkn/README.md"
  - "../rwdkn/README.md"
  - "../rwdkn-data-pipeline/README.md"
  - "../rwdkn-service/README.md"
---

# Knowledge Graph (KG) Documentation Hub

Welcome to the centralized documentation repository for the Knowledge Graph (KG) system. This documentation is structured using the **Open Knowledge Format (OKF)**, making it highly indexable and queryable for AI agents, while remaining clean and readable for humans.

## System Architecture & Specifications
- [RWDKN Repository Split Design Spec](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/repository-split.md) — Details on splitting the monolithic RWDKN into pipeline and service repositories.
- [Shared Infrastructure Package Namespace Refactoring](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/package-rename.md) — Details on refactoring `shared-infra` from `us.kolac.rkg` to `us.kolac.kg` and corresponding coordination/deployment pipeline enhancements.
- [Twelve-Factor App Configuration Compliance](file:///home/chandrakola/development/NLM/kg/kg-docs/architecture/twelve-factor-compliance.md) — Details on parameterizing `ckn`, `gkn`, `hphyskn`, and `nlmkn` environments to separate code from config.

## Workspace Directories & Projects
- [ckn](file:///home/chandrakola/development/NLM/kg/ckn) — Clinical Knowledge Network (Simulated fixture)
- [gkn](file:///home/chandrakola/development/NLM/kg/gkn) — General Knowledge Network (Simulated fixture)
- [hphyskn](file:///home/chandrakola/development/NLM/kg/hphyskn) — Human Physiology Knowledge Network (Simulated fixture)
- [nlmkn](file:///home/chandrakola/development/NLM/kg/nlmkn) — NLM-KN platform (Federation Hub + Participant Registry)
- [rwdkn](file:///home/chandrakola/development/NLM/kg/rwdkn) — Real World Data Knowledge Network
- [rwdkn-data-pipeline](file:///home/chandrakola/development/NLM/kg/rwdkn-data-pipeline) — Data pipeline for RWDKN
- [rwdkn-service](file:///home/chandrakola/development/NLM/kg/rwdkn-service) — Microservices for RWDKN
- [shared-infra](file:///home/chandrakola/development/NLM/kg/shared-infra) — Shared infrastructure configuration and VMs

## Formatting Standards
All documentation in this repository follows the Open Knowledge Format:
1. Every file is written in Markdown (`.md`).
2. Every file begins with a YAML frontmatter block defining its type, title, description, and tags.
3. Links to related concepts are listed in the frontmatter `links` block and inline.

You can find templates for creating new OKF-compliant documentation in [templates/okf-template.md](file:///home/chandrakola/development/NLM/kg/kg-docs/templates/okf-template.md).
