Yaml files for homelab Kubernetes cluster.   Using infrastructure as code and GitOps.

Workflow is a little different than typical Git workflows, where instead of having different branches for different environments (ie. main branch--> production, staging branch --> testing, develop --> development, etc...), all environments are available from the main branch, but from different subdirectories (ie. apps/staging dir applies to testing environments, production dir to production environments).  This allows for shared base files while having separate staging and production files without having to manage multiple branches.  This could change over time, depending on how the flow performs.

This project is broken down into 3 areas: Applications, Infrastructure and Monitoring.

# Applications
The following applications have so far been deployed:

## Linkding
A bookmark manager.  https://linkding.link

## Audiobookshelf
An audiobook and podcast media file manager.  https://www.audiobookshelf.org/

# Infrastructure

## FluxCD
A CI/CD solution that allows code changes to be applied to the cluster, once applied to the main branch/trunk.  This is the base of the GitOps which bootstraps the workflow by cluster (production, staging, etc...), and this is why it is installed in its own clusters directory.  https://fluxcd.io/

## Mend Renovate
An application that monitors for new releases of applications and creates Pull Requests automatically for review.  An update to a new release on the cluster becomes as simple as merging the Pull Request into the main branch/trunk.  https://www.mend.io/mend-renovate/

# Monitoring

## Prometheus + Grafana
Installed as a stack for metrics, dashboards and alerts.  Prometheus takes care of scraping and storing metrics, while Grafana visualizes them via Dashboards.  Alermanager can route notifications to external systems like email or Slack.  https://github.com/prometheus-community/helm-charts/releases/tag/kube-prometheus-stack-82.15.1
