# Kubernetes Learning Lab

A hands-on, code-first path to learning Kubernetes — every concept paired with real YAML manifests and `kubectl` commands run against a local `kind` cluster.

See [`k8s.md`](./k8s.md) for the full 10-module curriculum:

1. Pods — the smallest deployable unit
2. Deployments — self-healing & scaling
3. Services — stable networking
4. ConfigMaps & Secrets — externalized config
5. Volumes — persisting data
6. Namespaces — isolating environments
7. Ingress — routing HTTP traffic
8. Health checks — readiness/liveness probes
9. Helm — packaging manifests
10. Bringing in a QA/test background — CI integration, chaos-lite testing

## Setup

```bash
brew install kind kubectl
kind create cluster --name learning
kubectl get nodes
```

## Progress

- [x] Module 1 — Pods (`pod.yaml`)
- [x] Module 2 — Deployments (`deployment.yaml`)
- [x] Module 3 — Services (`service.yaml`)
- [x] Module 4 — ConfigMaps & Secrets (`config.yaml`)
- [x] Module 5 — Volumes (`volumes.yaml`)
- [x] Module 6 — Namespaces (`namespaces.yaml`)
- [x] Module 7 — Ingress (`ingress.yaml`, `kind-ingress-cluster.yaml`)
- [ ] Module 8 — Health checks
- [ ] Module 9 — Helm
- [ ] Module 10 — CI integration
