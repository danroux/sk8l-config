# sk8l-config

ArgoCD configuration for deploying [sk8l](https://sk8l.io) with its Helm chart.
Code lives in [sk8l-api](https://github.com/danroux/sk8l-api) and
[sk8l-ui](https://github.com/danroux/sk8l-ui); this repo only holds what runs
in the cluster.

## Layout

```
bootstrap/          applied by hand, once
  project.yaml        AppProject "sk8l" (allowed sources and namespaces)
  root.yaml           app-of-apps: syncs everything under apps/
  argocd-cm-health-patch.yaml
apps/               one ArgoCD Application per component, ordered by sync wave
  cert-manager.yaml   wave -3
  trust-manager.yaml  wave -2
  sk8l-pki.yaml       wave -1  -> manifests/sk8l-pki
  sk8l.yaml           wave  0  -> sk8l chart + values/sk8l/values.yaml
manifests/sk8l-pki/ issuers, certificates and trust bundle for sk8l's TLS secrets
values/sk8l/        chart values; image tags are bumped here by CI
```

## Bootstrap

Prerequisites: ArgoCD 2.6+ installed in the `argocd` namespace.

```sh
# 1. Let sync waves wait on child Application health (one-time)
kubectl -n argocd patch configmap argocd-cm --type merge \
  --patch-file bootstrap/argocd-cm-health-patch.yaml

# 2. Only if this repo is private: give ArgoCD read access
argocd repo add git@github.com:danroux/sk8l-config.git --ssh-private-key-path <key>
#    (and switch the repoURLs in bootstrap/ and apps/ to the SSH URL)

# 3. Create the project and the root app; everything else follows from git
kubectl apply -f bootstrap/project.yaml
kubectl apply -f bootstrap/root.yaml
```

## Access

```sh
kubectl -n sk8l port-forward svc/sk8l-ui 8001:8001 9080:9080
# https://localhost:8001
```

## Deploying a specific version

Edit `values/sk8l/values.yaml` and push to `master`. See the comments there for running
unreleased `pre-<sha>` builds from ghcr.
