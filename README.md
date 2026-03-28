Yaml files for homelab Kubernetes cluster.   Using infrastructure as code and GitOps.

Project is broken down into 3 areas: Applications, Infrastructure and Monitoring.

# Applications
The following applications have so far been deployed:

## Linkding
A bookmark manager.

## Audiobookshelf
An audiobook and podcast media file manager.

# Infrastructure

## FluxCD
A CI/CD solution that allows code changes to be applied to the cluster, once applied to the main branch/trunk.  This is the base of the GitOps which bootstraps the workflow, and this is why it is installed in its own cluster directory.

## Renovate
An application that monitors for new releases of applications and creates Pull Requests automatically for review.  An update to a new release on the cluster becomes as simple as merging the Pull Request into the main branch/trunk.

# Monitoring

## Prometheus + Grafana
Installed as a stack for metrics, dashboards and alerts.  Prometheus takes care of scraping and storing metrics, while Grafana visualizes them via Dashboards.  Alermanager can route notifications to external systems like email or Slack.
