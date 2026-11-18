# AGENTS.md

## Pre-requisites

Tools: `git gh kubectl flux kustomize sops age`

```bash
kubectl get nodes          # kubeconfig must reach the k3s cluster
gh auth status             # needs push access to this repo
ls ~/age.agekey            # sops age private key (not in ~/.config/sops)
```

## Agent setup

Run at session start; stop and tell the user if any fail.

```bash
export SOPS_AGE_KEY_FILE=~/age.agekey
kubectl config current-context && kubectl get nodes
flux check && flux get kustomizations
gh auth status
git fetch origin && git status
```

## Workflow

Flux applies `main`. Never commit to `main`, and never `kubectl apply`/`edit` Flux-managed resources (Flux reverts them).

```bash
git switch -c <branch> origin/main
kustomize build <overlay-dir> > /dev/null   # render what you touched
gh pr create --base main
```

The user merges, then says "merged". Then apply immediately:

```bash
flux reconcile source git flux-system
flux reconcile kustomization <name>   # apps | infrastructure-controllers | monitoring-controllers | monitoring-configs
flux get kustomizations && kubectl -n <ns> get pods
```

## Secrets

Never ask for secrets in chat or print their values. Have the user write them to a file outside the repo (e.g. `~/<app>-credentials.env`), then build the Secret with a script that does not echo values.

```bash
sops --encrypt --in-place --config clusters/staging/.sops.yaml <secret>.yaml
sops <secret>.yaml                                   # edit existing (needs SOPS_AGE_KEY_FILE)
kubectl -n <ns> rollout restart deploy/<name>        # pods don't reload env-var secrets
```

## Gotchas

- The League Tracker app code is in `Mackeul/theleaguetracker`; its default branch is `master`, not `main`.
- Reading the Grafana admin password is blocked by the permission classifier. To verify a dashboard loaded: `kubectl -n monitoring logs deploy/kube-prometheus-stack-grafana -c grafana-sc-dashboard`.
