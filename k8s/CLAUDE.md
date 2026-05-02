# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Apply manifests to a cluster

```bash
# Apply the full local overlay (recommended during development)
kubectl apply -k overlays/local/

# Apply only a specific namespace group
kubectl apply -k overlays/local/app/
kubectl apply -k overlays/local/messaging/
kubectl apply -k overlays/local/observability/

# Preview rendered manifests without applying
kubectl kustomize overlays/local/
```

### Validate manifests

```bash
# Dry-run apply to catch errors without touching the cluster
kubectl apply -k overlays/local/ --dry-run=client

# Render and pipe to kubeval or kubeconformant for schema validation
kubectl kustomize overlays/local/ | kubeconformant
```

### Inspect deployed resources

```bash
kubectl get all -n apps
kubectl get all -n messaging
kubectl get all -n observability
kubectl get networkpolicies -A
```

## Architecture

This repository migrates a microservices application into Kubernetes using **Kustomize** with a three-layer structure.

### Layers

```
base/          → raw manifests, no environment specifics
Components/    → reusable patches applied via Kustomize components
overlays/      → environment-specific composition (local, staging, prod)
```

### Namespaces and workloads

Resources are split across three namespaces that map to the overlay sub-directories:

| Namespace       | Workloads                                    |
|-----------------|----------------------------------------------|
| `apps`          | auth-api, users-api, todos-api, frontend, ingress |
| `messaging`     | redis-queue, log-processor                   |
| `observability` | zipkin-server                                |

The `base/` layer defines a single default `microservices` namespace and all raw deployments/services. The overlays reassign namespace via the Kustomize `namespace:` field, overriding what is declared in `base/`.

### How overlays compose resources

`overlays/local/` does **not** reference `base/` directly. Instead it delegates to three sub-kustomizations (app/, messaging/, observability/), each of which points to specific base manifests and sets the target namespace. The top-level `overlays/local/kustomization.yaml` only lists those three directories as resources.

`overlays/local/app/kustomization.yaml` is the most complex: it applies two Kustomize **components** (network-policies and resource-limits/local) and a replica-count patch from `patches/replicas-local.yaml` that forces all deployments to 1 replica.

### Kustomize components

`Components/` contains reusable sets of patches declared as `kind: Component` (API version `kustomize.config.k8s.io/v1alpha1`), which allows them to be included from multiple overlays without duplicating YAML.

- **network-policies** — NetworkPolicy manifests implementing a zero-trust model: every namespace starts with a deny-all ingress/egress policy; allow-rules then open only the traffic paths the application needs.
- **resource-limits/local** — Strategic merge patches that inject `resources.requests` and `resources.limits` into each Deployment for local development sizing. The `staging/` and `prod/` subdirectories exist but are currently empty.

### Network security model

Traffic is explicitly allowlisted. The general flow is:

```
ingress-nginx → frontend (apps)
frontend      → auth-api (apps)
auth-api      → users-api (apps)
todos-api     → redis-queue (messaging)
log-processor ← (receives from apps services)
apps services → zipkin-server (observability)
all pods      → kube-dns :53 (egress allowed)
```

All other cross-namespace and cross-pod traffic is dropped by the deny-all NetworkPolicies.

### Extending to staging/prod

`overlays/staging/` and `overlays/prod/` are stubs. To promote an environment, add its sub-kustomizations, point them at the appropriate `base/` manifests, populate `Components/resource-limits/staging|prod/`, and add environment-specific patches (image tags, replica counts, HPA, etc.).
