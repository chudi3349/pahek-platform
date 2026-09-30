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

**Target:** daily full backup (database + files), 30 days retained, copied to storage registered to PAHEK, report after each run. *Not yet in place — scheduled for build weeks 3–4. Until then there is no off-server copy.*

Manual backup:

```bash
$DC exec backend bench --site $SITE backup --with-files
```

Files land in the `sites` volume under `<site>/private/backups/`.

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
