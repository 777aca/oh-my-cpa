# OMC trial beside an existing CPA

Console: <https://cpa.axoai.cc/omc/>. Sign in with the existing CPA management key.

This deployment uses upstream OMC v0.4.3 with a separate SQLite volume and the existing
CPA Docker network. The application runs as `10001:10001`, with a read-only root filesystem
and no Docker socket. Updates are manual.

## Host layout

- Deployment directory: `/opt/oh-my-cpa`
- Private configuration: `/opt/oh-my-cpa/.env` (root-owned, mode `0600`)
- Container: `oh-my-cpa-trial`
- Data volume: `oh-my-cpa-trial-data`
- Host bind: `127.0.0.1:18317`, container port `8080`
- Existing external network: `cpa_default`
- Proxy fragment: `nginx-location.conf`, inserted into the existing HTTPS virtual host

The proxy preserves `/omc` and forwards streaming responses without buffering. On this
host, nginx runs inside `/srv/legacy-root`; validate configuration using
`sudo chroot /srv/legacy-root /www/server/nginx/sbin/nginx -t`, then gracefully reload it
with `sudo systemctl reload nginx`.

## Trial settings

Usage collection uses `subscribe`, which receives new events without draining CPA's
usage queue. Historical events from another panel are not imported. Scheduled quota reads,
post-reset automatic reads, OMC plugin-source reconciliation and automatic update checks
are disabled for this trial. Manual quota refresh remains available. Model warmup has no
configured tasks.

The login, credential list, quota refresh, plugin list, configuration read and OMC preference
write paths were checked against the existing gateway. No production configuration save,
credential status change, model inference or reset-credit redemption was performed.

## Operate

Run these commands on the Docker host:

```sh
cd /opt/oh-my-cpa
sudo docker compose ps
sudo docker compose stop oh-my-cpa
sudo docker compose up -d oh-my-cpa
curl -fsS https://cpa.axoai.cc/omc/api/healthz
```

A successful health response includes `status: ok`, `database_status: ok`,
`cpa_connected: true` and `cpa_management_api: v8`.

Before changing versions, back up the database and preserve the existing master key.
Follow the upstream [backup runbook](../../docs/ops/sqlite-operations.md); do not copy a live
SQLite WAL directory. Change the image version in `compose.yml`, pull that version and
recreate only the OMC service. Do not regenerate `.env` during an update.

Stopping OMC leaves the existing CPA and its original management panel running. Keep the
data volume when stopping the trial. The pre-trial nginx configuration is retained privately
on the host as `/opt/oh-my-cpa/nginx-before.conf`.
