# Architecture

## Runtime

```
Internet ──► host nginx (80/443, certbot TLS) ──► 127.0.0.1:8080 ──► frontend container
                                                                      ├─ backend (gunicorn)
                                                                      ├─ websocket
                                                                      ├─ queue-short / queue-long / scheduler
                                                                      ├─ db (MariaDB 11.8)
                                                                      └─ redis-cache / redis-queue
```

- Docker Compose project `pahek`, built from upstream `frappe_docker` plus three overrides: `compose.mariadb.yaml`, `compose.redis.yaml` (both upstream) and `compose.pahek-limits.yaml` (ours, in this repo).
- Custom image `pahek-erpnext:<tag>` contains Frappe, ERPNext and PAHEK's apps, all listed in `apps.json`.
- **DNS multitenancy:** the frontend picks the site from the incoming `Host` header. Host nginx must forward it unchanged (`proxy_set_header Host $host;`). A new site = `bench new-site` + DNS record + nginx server block; no rebuild.
- No Traefik. Host nginx is the only edge. Port 8080 is bound to localhost only.

## Memory budget (8 GB host, 12 GB swap as a safety net only)

| Container | `mem_limit` |
|---|---|
| backend | 1200m |
| db (InnoDB buffer pool 1G) | 1800m |
| frontend, queue-short, queue-long, redis-cache, redis-queue | 300m each |
| websocket, scheduler, configurator | 200m each |

Caps total ~4.9 GB; real usage is far lower. Headroom is reserved for The Message Refinery (WordPress stack, ~300–500 MB) and, if chosen, a self-hosted Matomo (~300–500 MB). Measure with `docker stats --no-stream` before adding anything else.

## Apps

| App | Contains | Install on |
|---|---|---|
| `pahek_core` | `PAHEK Site Settings` (organisation identity), roles `PAHEK Admin`, `PAHEK Editor`, `AI Drafter` | Every PAHEK site |
| `pahek_web` | PAHEK's own website content, enquiries, SEO, AI-draft rules | **PAHEK's own site only** |

Reserved names, not built: `pahek_sms` (Phase 2 Security Management System), `pahek_campaigns` (campaign engine), `pahek_lms` (Phase 3). The split exists so a future SMS sold to other firms never ships PAHEK's website code or content.

**Naming rule:** every custom DocType is prefixed `PAHEK` and checked against the site's full DocType list before creation. Frappe core already has `Portal Settings`, `Contact Us Settings`, `About Us Settings` etc.; an unprefixed name risks overwriting core metadata on migrate.

## Roles and the AI-draft rule

| Role | Who | Can make content public |
|---|---|---|
| PAHEK Admin | Charles | Yes |
| PAHEK Editor | Staff Charles appoints | Yes |
| AI Drafter | The dedicated MCP service user only | **No** |

Content goes public through a `published` field. That field is placed at permlevel 1; only PAHEK Admin/Editor hold write at permlevel 1. The AI connects as its own user holding only `AI Drafter`, so Frappe itself rejects any attempt by the AI to publish. **The AI connection must never authenticate as Charles or as Administrator** — that would bypass the rule entirely.

ETLS maintenance access during the build and the 60-day post-launch support window is a named user account with System Manager, disabled at the end of that window. It is not a role.

## Backups

Daily full backup (database + files), 30-day retention, copied off-server to storage registered to PAHEK, with a report after each run. See RUNBOOK.md.
