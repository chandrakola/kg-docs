---
type: spec
title: "Twelve-Factor App Configuration Compliance"
description: "Specification of the environment configuration decoupling applied to ckn, gkn, hphyskn, and nlmkn to separate code from network environment profiles."
tags: [architecture, twelve-factor, environment, config, docker, nginx]
version: 1.0.0
created: 2026-07-16
updated: 2026-07-16
links:
  - "../README.md"
  - "../../ckn/deploy/docker-compose.kn.yml"
  - "../../nlmkn/auth/app.py"
---

# Twelve-Factor App Configuration Compliance

This document records the refactoring changes applied across the Knowledge Graph workspace to satisfy **Factor III ("Store config in the environment")** of the Twelve-Factor App principles, removing hardcoded domains and enabling clean execution on `localhost` or any custom DNS.

---

## 1. Parameterized Ingress Routing (Traefik)

*   **Before:** Traefik routers in `docker-compose` declared hardcoded domains like `Host(ckn.kolac.us)` or `Host(nlmkn.kolac.us)`.
*   **After:** Subdomain hosts are defined using dynamic environment variables with safe defaults:
    *   `ckn` Host: `${CKN_HOST:-ckn.localhost}`
    *   `gkn` Host: `${GKN_HOST:-gkn.localhost}`
    *   `hphyskn` Host: `${HPHYSKN_HOST:-hphyskn.localhost}`
    *   `nlmkn` (Hub & Auth) Host: `${HUB_HOST:-nlmkn.localhost}`
*   **Local Portability:** Because any domain ending in `.localhost` automatically resolves to `127.0.0.1` in modern browsers, developers can launch and test the entire ecosystem locally without modifying `/etc/hosts`.

---

## 2. Dynamic CORS Origins in Nginx Mocks

*   **Before:** Static configuration files (`edge.kn.nginx.conf`) had hardcoded `Access-Control-Allow-Origin: https://nlmkn.kolac.us` headers, blocking local cross-origin queries.
*   **After:**
    *   Renamed config files to `edge.kn.nginx.conf.template`.
    *   Mounted them in the container under `/etc/nginx/templates/default.conf.template:ro`. On startup, Nginx dynamically replaces the variable `${HUB_ORIGIN}` with the value of the environment variable (defaulting to `http://nlmkn.localhost`).

---

## 3. Decoupled SSO Auth Gateway

*   **Before:** The SSO forward authentication app (`nlmkn/auth/app.py`) had hardcoded `.kolac.us` defaults for redirects, cookie domains, and security checks.
*   **After:**
    *   `SSO_COOKIE_DOMAIN` defaults to `.localhost` (so it scopes across all local subdomains).
    *   `SSO_ALLOWED_DOMAINS` reads a comma-separated list of safe redirect locations (defaulting to `.kolac.us,.localhost,localhost`).
    *   `SSO_LOGIN_URL` and `SSO_DEFAULT_RD` default to relative paths (`/__auth/login` and `/hub/` respectively), letting Flask compute host redirects dynamically based on incoming headers.
    *   Replaced absolute path `env_file: /run/nlmkn/sso.env` inside `docker-compose.hub.yml` with direct `environment` blocks for container orchestration.
