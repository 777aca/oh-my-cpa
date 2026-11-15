# OMC at the existing CPA origin

Console: <https://cpa.axoai.cc/>. Sign in with the existing CPA management key.

This deployment uses upstream OMC v0.4.3 with a separate SQLite volume and the existing
CPA Docker network. The official CPA gateway remains a separate service. The console runs
as `10001:10001`, with a read-only root filesystem and no Docker socket. Updates are manual.

## Host layout

- Deployment directory: `/opt/oh-my-cpa`
- Private configuration: `/opt/oh-my-cpa/.env` (root-owned, mode `0600`)
- Container: `oh-my-cpa-trial`
- Data volume: `oh-my-cpa-trial-data`
- Host bind: `127.0.0.1:18317`, container port `8080`
- Existing external network: `cpa_default`
- Proxy fragment: `nginx-location.conf`, inserted into the existing HTTPS virtual host

The proxy serves OMC at `/`, including its `/api` endpoints and embedded assets. The old
`/management.html` and `/omc/` URLs redirect to the root. CPA's versioned gateway APIs,
plugin namespaces, health endpoint and OAuth callbacks continue to reach port `8317`;
the existing BPS batch locations remain in the surrounding virtual host. Check these
gateway routes when upgrading CPA or adding a custom HTTP extension.

The container and data volume keep their original trial names to retain existing data.
Streaming responses are forwarded without buffering. On this host, nginx runs inside
`/srv/legacy-root`; validate configuration using
`sudo chroot /srv/legacy-root /www/server/nginx/sbin/nginx -t`, then gracefully reload it
with `sudo systemctl reload nginx`.

## Runtime settings

Usage collection uses `subscribe`, which receives new events without draining CPA's
usage queue. Historical events from another panel are not imported. Scheduled quota reads,
post-reset automatic reads, OMC plugin-source reconciliation and automatic update checks
remain disabled. Manual quota refresh remains available. Model warmup has no
configured tasks.

CPA's `observability.usage.usage-statistics-enabled` is enabled so it emits request
accounting events. The usage subscription can be healthy while this setting is disabled;
verify that captured, decoded and stored event counts increase after a real request.
Enable it through OMC's configuration panel to retain the audited pre-write backup. The
setting applies without restarting CPA. Requests made while statistics were disabled
cannot be reconstructed from the subscription.

The login, credential list, quota refresh, plugin list, configuration read and OMC preference
write paths were checked against the existing gateway. The usage statistics flag was
enabled through the audited scalar configuration API. Credential status changes, model
inference and reset-credit redemption were not part of acceptance.

Root-path acceptance checked login/logout, account and configuration reads, desktop and
320px/390px quota layouts, old URL redirects, and gateway route parity. The existing
`cpa-bps-0534` page resource returns `404` both directly from CPA and through OMC's plugin
host. Its page needs separate investigation; the BPS batch service health endpoint works.

## Codex Candy Eval on WebKit

The installed v0.4.6 plugin copies the console's CSS-connected font faces into its own
document. WebKit throws `InvalidModificationError` for this transfer, interrupting startup
before either the login form or plugin body appears. The exact plugin UI location in
`nginx-location.conf` treats only this optional font-transfer exception as non-fatal.
The request still passes through OMC's authenticated plugin host; API routes and other
exceptions are unchanged. Upstream compression is disabled for this HTML response so
nginx can apply the compatibility replacement before compressing the client response.

Acceptance reproduced the blank page in WebKit, then verified the same hosted page after
the one-statement replacement. Remove this location when an upstream plugin update fixes
its font transfer, after repeating WebKit and Chromium acceptance. The pre-fix proxy backup
is `/opt/oh-my-cpa/nginx-before-candy-webkit-20261115.conf`.

## Operate

Run these commands on the Docker host:

```sh
cd /opt/oh-my-cpa
sudo docker compose ps
sudo docker compose stop oh-my-cpa
sudo docker compose up -d oh-my-cpa
curl -fsS https://cpa.axoai.cc/api/healthz
```

A successful health response includes `status: ok`, `database_status: ok`,
`cpa_connected: true` and `cpa_management_api: v8`.

Before changing versions, back up the database and preserve the existing master key.
Follow the upstream [backup runbook](../../docs/ops/sqlite-operations.md); do not copy a live
SQLite WAL directory. Change the image version in `compose.yml`, pull that version and
recreate only the OMC service. Do not regenerate `.env` during an update.

Stopping OMC leaves CPA's gateway APIs running; the root console requires OMC. Keep the
data volume when stopping the console. The original management HTML remains on disk.

## Restore the previous URL layout

The root-path migration's Compose and nginx backups are retained privately in
`/opt/oh-my-cpa/root-migration-20261115/`. To restore the previous `/omc/` layout, restore
`compose.yml` to the deployment directory and `nginx.conf` to
`/www/server/panel/vhost/nginx/cpa.axoai.cc.conf`. Recreate only the OMC service, validate
nginx using the chroot command above, then reload nginx. Preserve `.env` and the data volume.
The earlier pre-trial proxy backup is `/opt/oh-my-cpa/nginx-before.conf`.
