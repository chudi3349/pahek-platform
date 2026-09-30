# Deploy

All commands run on `pahek-server` as the `frappe` user, from `/home/frappe/pahek/frappe_docker`, unless stated otherwise.

Shorthand used below:

```bash
DC="docker compose --project-name pahek --env-file .env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.pahek-limits.yaml"
SITE=pahek.ejitech.life
```

Always pass all four `-f` files to `up`. Running `up` with fewer files silently drops the memory limits and the localhost-only port binding.

## 1. Back up first

```bash
$DC exec backend bench --site $SITE backup --with-files
```

Note the backup file names printed; they are the rollback point.

## 2. Build the image (only when app code or `apps.json` changed)

The private app repos need a read-only access token at build time. Make a build copy of `apps.json` with the token in the URLs, pass it as a **BuildKit secret**, then delete it:

```bash
# apps.build.json = apps.json with https://<token>@github.com/... URLs. Never commit it.
docker build --no-cache \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --secret=id=apps_json,src=apps.build.json \
  --tag=pahek-erpnext:<new-tag> \
  --file=images/layered/Containerfile .
shred -u apps.build.json
```

- Never use `--build-arg=APPS_JSON_BASE64=...`. Current frappe_docker ignores it without an error and produces an image with no apps. Build args are also stored in the image, which would leak the token.
- Use a new tag every build (e.g. `v16-2026.10.02`). Keep the previous tag; it is the rollback image.
- Confirm the apps are really in the image before going further:

```bash
docker run --rm pahek-erpnext:<new-tag> ls /home/frappe/frappe-bench/apps/
```

## 3. Switch and migrate

Set `CUSTOM_TAG=<new-tag>` in `.env`, then:

```bash
$DC up -d
$DC exec backend bench --site $SITE migrate
```

First install of a new app: `$DC exec backend bench --site $SITE install-app <app>` before `migrate`.

## 4. Verify

- `$DC ps` — all services up, `db` healthy
- `curl -sI https://$SITE | head -1` — `200`
- Log in to the Desk; open one page and one PAHEK DocType
- Once built: submit the enquiry form as an anonymous visitor and confirm a Lead appears; confirm the AI Drafter user cannot publish (see RUNBOOK.md)

## 5. Roll back

1. Set `CUSTOM_TAG` back to the previous tag, `$DC up -d`.
2. If `migrate` changed the database, restore the backup from step 1:

```bash
$DC exec backend bench --site $SITE restore <db-backup.sql.gz> \
  --with-public-files <files.tar> --with-private-files <private-files.tar>
```

## Changing only the compose override

Copy the new `overrides/compose.pahek-limits.yaml` from this repo into the checkout, then:

```bash
$DC config -q      # must print nothing
$DC up -d          # recreates only the services whose settings changed
```
