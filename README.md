Yaml files for homelab Kubernetes cluster.   Using infrastructure as code and GitOps.

Workflow is a little different than typical Git workflows, where instead of having different branches for different environments (ie. main branch--> production, staging branch --> testing, develop --> development, etc...), all environments are available from the main branch, but from different subdirectories (ie. apps/staging dir applies to testing environments, production dir to production environments).  This is called the Monorepo approach and allows for shared base files while having separate staging and production files without having to manage multiple branches.  This could change over time, depending on how the flow performs.  See https://fluxcd.io/flux/guides/repository-structure/

This project is broken down into 3 areas: Applications, Infrastructure and Monitoring.

# Applications
The following applications have so far been deployed:

## Linkding
A bookmark manager.  https://linkding.link

## Audiobookshelf
An audiobook and podcast media file manager.  https://www.audiobookshelf.org/

## The League Tracker
Web based sports league statistics tracking.  https://theleaguetracker.com

Runs as a single replica (nginx + PHP-FPM in one image from `ghcr.io/mackeul/theleaguetracker`) with its SQLite database and logo files on a local-path volume.  It is reached from the internet through a Cloudflare tunnel (`theleaguetracker`), with `theleaguetracker.com` and `www` as proxied CNAMEs to the tunnel.  Mail (sign-up confirmation, password reset, invitations) is sent through Proton Mail SMTP from `admin@theleaguetracker.com`.

- **Upgrading**: bump `newTag` under `images:` in `apps/staging/theleaguetracker/kustomization.yaml`.  That one line sets the version for both the app and the backup job.  The deployment uses the Recreate strategy, so there is a short outage while migrations run on start.
- **Settings**: non-secret settings are in `apps/base/theleaguetracker/config.env` (`KEY=value`, no quotes).  The ConfigMap is generated with a content hash, so a change restarts the pod automatically.  Secrets (`APP_KEY`, SMTP credentials) are sops-encrypted in `theleaguetracker-secrets.yaml`; after changing them, run `kubectl -n theleaguetracker rollout restart deploy/theleaguetracker`.
- **Demo league**: "Riverside Rec Basketball" was seeded by hand with `php artisan db:seed --class=DemoSeeder --force`.  It comes with the shared sign-in `demo@example.com` / `password`, so anyone can edit the demo data.

### Backups
A CronJob (`theleaguetracker-backup`) runs nightly at 03:30 America/Toronto:

1. The app image takes a consistent copy of the live database (`VACUUM INTO`) and tars it together with the logos.
2. rclone uploads the tarball to the Cloudflare R2 bucket `theleaguetracker-backups` and deletes backups older than 30 days.

The R2 API token (Object Read & Write, scoped to that bucket) is sops-encrypted in `apps/staging/theleaguetracker/backup-r2-secret.yaml`.  A `TheLeagueTrackerBackupStale` alert fires if no backup has succeeded in 26 hours.

Run a backup now:

```bash
kubectl -n theleaguetracker create job --from=cronjob/theleaguetracker-backup backup-manual
kubectl -n theleaguetracker logs job/backup-manual -c upload
kubectl -n theleaguetracker delete job backup-manual
```

Restore: download a tarball from the bucket, scale the deployment to 0, copy `database.sqlite` and `public/` back into the volume (mounted at `/var/www/html/storage/app` in the app), then scale back to 1.  The steps are also in the comments at the top of `apps/staging/theleaguetracker/backup.yaml`.

# Infrastructure

## FluxCD
A CI/CD solution that allows code changes to be applied to the cluster, once applied to the main branch/trunk.  This is the base of the GitOps which bootstraps the workflow by cluster (production, staging, etc...), and this is why it is installed in its own clusters directory.  https://fluxcd.io/

## Mend Renovate
An application that monitors for new releases of applications and creates Pull Requests automatically for review.  An update to a new release on the cluster becomes as simple as merging the Pull Request into the main branch/trunk.  https://www.mend.io/mend-renovate/

# Monitoring

## Prometheus + Grafana
Installed as a stack for metrics, dashboards and alerts.  Prometheus takes care of scraping and storing metrics, while Grafana visualizes them via Dashboards.  Alertmanager can route notifications to external systems like email or Slack.  https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

Grafana is at https://grafana.malazan.wu.

### Adding an app to monitoring
- **Scraping**: Prometheus only picks up `ServiceMonitor`/`PodMonitor` and `PrometheusRule` resources labelled `release: kube-prometheus-stack`, in any namespace.
- **Dashboards**: the Grafana sidecar loads any ConfigMap labelled `grafana_dashboard: "1"`, in any namespace.  Keep the dashboard JSON in the app's directory and generate the ConfigMap with a `configMapGenerator` (see `apps/staging/theleaguetracker/kustomization.yaml`).
- **Alert labels**: give each alert rule a `namespace` label.  Aggregations like `sum()` drop it, and Alertmanager routes on it.

### The League Tracker
`apps/staging/theleaguetracker/monitoring.yaml` and `dashboards/theleaguetracker.json` define:

- A `PodMonitor` for cloudflared (`:2000/metrics`).  The app has no metrics endpoint of its own, and every request comes through the tunnel, so the tunnel's request counts by status code measure real traffic.
- The "The League Tracker" dashboard: status (app up, tunnel connections, requests, 5xx rate, last backup, restarts), traffic by status code, and CPU, memory and network.
- Alerts:
  - critical: `TheLeagueTrackerDown`, `TheLeagueTrackerTunnelDown` (5 minutes)
  - warning: `TheLeagueTrackerServerErrors` (more than 5% 5xx), `TheLeagueTrackerBackupStale` (no backup in 26 hours), `TheLeagueTrackerMemoryHigh` (more than 90% of the limit)

### Alert email
Alertmanager sends email through Proton SMTP (`smtp.protonmail.ch:587`) from `admin@theleaguetracker.com`.  The config is in the `alertmanager.config` values of `monitoring/controllers/base/kube-prometheus-stack/release.yaml`, and the SMTP token is sops-encrypted in `monitoring/configs/staging/kube-prometheus-stack/alertmanager-smtp.yaml`.

Only alerts with `namespace="theleaguetracker"` are emailed, to `admin@theleaguetracker.com`, with resolved notices and a reminder every 12 hours while an alert is still firing.  Every other alert, including `Watchdog`, goes to the `null` receiver and only shows in Grafana and Alertmanager.  To email another app's alerts, add a route for its namespace.

Known noise, not emailed: `KubeControllerManagerDown`, `KubeSchedulerDown` and `KubeProxyDown` always fire on k3s, which runs those components inside its own process where Prometheus can't scrape them.
