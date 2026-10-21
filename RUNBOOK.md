# Runbook

Uses the `$DC` / `$SITE` shorthand from DEPLOY.md.

## Health check

```bash
$DC ps
docker stats --no-stream
free -h && df -h /
$DC exec backend bench --site $SITE doctor
curl -sI https://$SITE | head -1
```

Warning sign: swap in heavy use (`free -h`). Swap is a shock absorber, not capacity — find the container that grew (`docker stats`) and cap or fix it.

## Backups

**Target:** daily full backup (database + files), 30 days retained, copied to storage registered to PAHEK, report after each run.

**Live since 2026-10-21:** `overrides/compose.backup-cron.yaml` adds an Ofelia-scheduled `cron` service running two jobs against the `scheduler` container:

- `datecron` — `bench --site all backup --with-files`, daily at 02:00 server time (`BACKUP_CRONSTRING` in `.env` overrides the schedule if it ever needs to change)
- `pruneold` — deletes backup files older than 30 days, daily at 03:00 (an hour after the backup, so it never races a run still in progress)

Add `-f overrides/compose.backup-cron.yaml` to every `up -d` from now on (DEPLOY.md's standard command needs this fifth file). Confirm it's running: `docker logs pahek-cron-1` should show both jobs registered with their schedules.

**Still open:** off-server copy. A backup still only exists on the server itself until Charles opens a Hetzner Storage Box (or equivalent) in PAHEK's name — see `sop/client/pahek-phase1-build-week1-backups-2026-10-21` for exactly what needs wiring in once that account exists (an `rclone`/`restic` sync job, same cron pattern, pushing the `private/backups/` directory off-server). **No "confirmation after each run" notification either** — same blocker as the enquiry form's email notification: no Email Account/Zoho SMTP is configured on this site at all yet.

Manual backup:

```bash
$DC exec backend bench --site $SITE backup --with-files
```

Files land in the `sites` volume under `<site>/private/backups/` — this is a named Docker volume (`VOLUME` in the Containerfile), so it survives container recreates and image rebuilds.

## Restore

See DEPLOY.md step 5. Test a restore against a scratch site at least once after the backup job is built, and after any major upgrade.

## TLS certificates

Certbot on the host renews automatically (`systemctl status certbot.timer`). Check: `sudo certbot certificates`. Test: `sudo certbot renew --dry-run`.

## SSH access

- Password login is disabled; keys only.
- `fail2ban` protects SSH (`sudo fail2ban-client status sshd`). To unban an address: `sudo fail2ban-client set sshd unbanip <ip>`.
- If locked out entirely: Hetzner Cloud Console → server → Console (browser-based, works without SSH).

## Common failures

| Symptom | Cause / fix |
|---|---|
| Image builds fine but has no apps | Built with `APPS_JSON_BASE64`. Rebuild with `--secret=id=apps_json,...` and `--no-cache` (DEPLOY.md step 2) |
| `WARN ... variable is not set` from compose | A `$` inside `.env`. Regenerate the value without `$` |
| Site shows the wrong site or 404 for every page | Host nginx not forwarding `Host $host` |
| Memory limits missing after a restart | `up` was run without all four `-f` files |
| Background jobs vanish | Someone set eviction on `redis-queue`. It must never have `maxmemory-policy` |
| `db` restarts in a loop after an override change | Bad MariaDB flag in `compose.pahek-limits.yaml` `command:`. `$DC logs db`, fix, `$DC up -d` |
| Disk filling up over time | `--no-cache` builds still *write* BuildKit cache even though they don't read it — it grows unbounded across repeated rebuilds (hit 86GB / 80% disk by 2026-10-21, 12 rebuilds in). Check with `docker system df`; clear with `docker builder prune -f`. Also remove old image tags you're not keeping as a rollback point (`docker images \| grep pahek-erpnext`, `docker rmi <tag>`) — each one is 4GB+. Do this check after every few rebuilds, not just when disk alerts fire. |
