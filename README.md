# PAHEK Platform

Infrastructure and documentation for PAHEK Security's Frappe/ERPNext platform, running on `pahek-server` (Hetzner CPX32, Ubuntu 24.04).

This repository is the single source of truth for how the platform is built and run. A competent developer with access to this repo, the two app repos and the server should be able to operate, rebuild or hand over the platform without anyone from the original build team.

## Repositories

| Repo | What it holds |
|---|---|
| `pahek-platform` (this one) | `apps.json`, compose overrides, and all platform documentation |
| `pahek_core` | Frappe app: organisation settings, roles |
| `pahek_web` | Frappe app: PAHEK's public website, content, enquiries, AI-draft rules |

Each Frappe app is its own repository because the Docker image build (`apps.json`) clones every app from its own git URL.

## Documents

| File | Read it when |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | You need to understand how the pieces fit and why |
| [DEPLOY.md](DEPLOY.md) | You are shipping a change to the server |
| [RUNBOOK.md](RUNBOOK.md) | Something is wrong, or you need to back up / restore / renew |
| [IMPLEMENTATION_NOTES.md](IMPLEMENTATION_NOTES.md) | You want to know why a decision was made (running log, newest last) |

## Where things live on the server

| Thing | Location |
|---|---|
| frappe_docker checkout (upstream, unmodified) | `/home/frappe/pahek/frappe_docker` |
| This repo's `apps.json` and `overrides/compose.pahek-limits.yaml` | copied into the frappe_docker checkout at deploy time |
| Environment file (secrets) | `/home/frappe/pahek/frappe_docker/.env` — never committed; values in Passbolt |
| Host nginx site configs | `/etc/nginx/conf.d/*.conf` |
| TLS certificates | `/etc/letsencrypt/` (certbot, auto-renewing) |

## Ownership

The server, domains, repositories and all accounts belong to PAHEK Security. Built by Eji Tech Life Services under proposal `PROP-2026-00003`.
