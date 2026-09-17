# Learning Kubernetes Through Code

A hands-on, code-first path to Kubernetes. Every section pairs a concept with real YAML/CLI you run yourself — no slide decks, just clusters.

---

## 0. Setup (do this first)

You need a local cluster, not a cloud account, to learn fast and free.

```bash
# Install kind (Kubernetes in Docker) — lightweight, disposable clusters
brew install kind          # macOS
# or: go install sigs.k8s.io/kind@latest

# Install kubectl (the CLI you'll live in)
brew install kubectl

# Create your first cluster
kind create cluster --name learning

# Confirm it's alive
kubectl get nodes
kubectl cluster-info
```

Optional but recommended: `k9s` (terminal UI for watching cluster state live) — `brew install k9s`.

---

## Module 1 — Pods: the smallest unit

A Pod wraps one or more containers that share network/storage.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    app: hello
spec:
  containers:
    - name: hello
      image: nginx:1.27
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod hello-pod
kubectl logs hello-pod
kubectl port-forward hello-pod 8080:80   # visit localhost:8080
kubectl delete -f pod.yaml
```

**Exercise:** Create a Pod running `redis:alpine`. Exec into it and run `redis-cli ping`.
```bash
kubectl exec -it <pod-name> -- redis-cli ping
```

---

## Module 2 — Deployments: self-healing & scaling

You rarely create bare Pods. Deployments manage replicas and rollouts.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: nginx:1.27
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods -l app=hello

# Kill a pod and watch Kubernetes replace it automatically
kubectl delete pod <one-of-the-pod-names>
kubectl get pods -w

# Scale
kubectl scale deployment hello-deploy --replicas=5

# Rolling update
kubectl set image deployment/hello-deploy hello=nginx:1.28
kubectl rollout status deployment/hello-deploy
kubectl rollout undo deployment/hello-deploy
```

**Exercise:** Break the image name on purpose (`nginx:doesnotexist`), watch `kubectl rollout status` hang, then roll back.

---

## Module 3 — Services: stable networking

Pods die and get new IPs constantly. A Service gives them a stable address.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f service.yaml
kubectl get svc
kubectl run tmp --rm -it --image=busybox -- wget -qO- hello-svc
```

Service types to know:
| Type | Use case |
|---|---|
| `ClusterIP` | internal-only (default) |
| `NodePort` | expose on each node's IP:port, for local testing |
| `LoadBalancer` | cloud provider provisions an external LB |

**Exercise:** Change `type` to `NodePort`, find the assigned port with `kubectl get svc`, and curl it from your host machine.

---

## Module 4 — ConfigMaps & Secrets

Externalize config instead of baking it into images.

```yaml
# config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "staging"
  LOG_LEVEL: "debug"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_PASSWORD: "supersecret123"
```

```yaml
# reference them in a pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
```

```bash
kubectl apply -f config.yaml
kubectl exec -it <pod> -- env | grep -E "APP_ENV|DB_PASSWORD"
```

**Exercise:** Update `LOG_LEVEL` in the ConfigMap, then confirm the running Pod does *not* auto-update (ConfigMaps aren't hot-reloaded by default) — this is a common real-world gotcha worth internalizing.

---

## Module 5 — Volumes: persisting data

Containers are ephemeral by default. Volumes survive container restarts.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  selector:
    matchLabels: { app: db }
  template:
    metadata:
      labels: { app: db }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              value: "postgres"
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: data-pvc
```

**Exercise:** Deploy this, write data into Postgres, delete the Pod (not the PVC), confirm the new Pod still has your data.

---

## Module 6 — Namespaces: isolating environments

```bash
kubectl create namespace staging
kubectl create namespace production
kubectl apply -f deployment.yaml -n staging
kubectl get pods -n staging
kubectl config set-context --current --namespace=staging   # stop typing -n every time
```

**Exercise:** Deploy the same manifests into both `staging` and `production` namespaces and confirm they don't interfere with each other.

---

## Module 7 — Ingress: routing HTTP traffic

```bash
# Install an ingress controller in kind
kind create cluster --name ingress-lab --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
EOF

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: hello.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-svc
                port:
                  number: 80
```

```bash
echo "127.0.0.1 hello.local" | sudo tee -a /etc/hosts
curl http://hello.local
```

---

## Module 8 — Health checks (crucial for QA mindset)

```yaml
containers:
  - name: hello
    image: nginx:1.27
    readinessProbe:
      httpGet: { path: /, port: 80 }
      initialDelaySeconds: 3
      periodSeconds: 5
    livenessProbe:
      httpGet: { path: /, port: 80 }
      initialDelaySeconds: 10
      periodSeconds: 10
    resources:
      requests: { cpu: "100m", memory: "64Mi" }
      limits: { cpu: "250m", memory: "128Mi" }
```

- **readinessProbe** — controls whether traffic is routed to the pod
- **livenessProbe** — controls whether Kubernetes restarts the pod
- **resources** — prevents one pod from starving the node

**Exercise:** Point the liveness probe at a path that 404s and watch Kubernetes repeatedly restart the container (`kubectl get pods -w`, check `RESTARTS` column).

---

## Module 9 — Helm: packaging manifests

```bash
brew install helm
helm create mychart
# edit mychart/values.yaml and mychart/templates/*.yaml

helm install my-release ./mychart
helm upgrade my-release ./mychart --set replicaCount=3
helm rollback my-release 1
helm uninstall my-release
```

**Exercise:** Templatize the `image.tag` and `replicaCount` from Modules 1–2 into a Helm chart, then deploy two different values files for staging vs production.

---

## Module 10 — Bringing in your QA/test background

Ways to connect Kubernetes to testing and automation, since that's your existing strength:

- **Ephemeral test environments**: spin up a `kind` cluster in CI (GitHub Actions has `kind-action`), deploy the app, run your Playwright/Jest suite against it, tear it down.
- **Testing rollout safety**: simulate a bad deployment (Module 2 exercise) and write a script that asserts `kubectl rollout status` fails within N seconds — useful as a CI gate.
- **Chaos-lite testing**: use `kubectl delete pod` mid-test-run to confirm your app's resilience/retry logic actually works.
- **Contract testing across services**: deploy two versions of a service side-by-side (different Deployments, same Service selector via labels) to test backward compatibility before a real rollout.

```yaml
# github actions snippet: spin up kind, deploy, test, teardown
- uses: helm/kind-action@v1
  with:
    cluster_name: ci-test
- run: kubectl apply -f k8s/
- run: kubectl wait --for=condition=available deployment/hello-deploy --timeout=60s
- run: npm run test:e2e -- --baseUrl=http://hello.local
```

---

## Suggested order & pacing

| Week | Focus |
|---|---|
| 1 | Modules 1–3 (Pods, Deployments, Services) — do every exercise twice |
| 2 | Modules 4–6 (Config, Volumes, Namespaces) |
| 3 | Modules 7–8 (Ingress, health checks) |
| 4 | Module 9 (Helm) + Module 10 (CI integration with your own test suite) |

## Reference commands cheat sheet

```bash
kubectl get all -A                     # everything, every namespace
kubectl describe <resource> <name>     # debug: events, status, config
kubectl logs -f <pod> -c <container>   # stream logs
kubectl exec -it <pod> -- sh           # shell into a container
kubectl explain deployment.spec        # inline API docs
kubectl diff -f file.yaml              # preview changes before apply
kubectl get events --sort-by=.lastTimestamp   # what just happened
```

## Further resources
- Official docs: https://kubernetes.io/docs/tutorials/
- `kubectl` interactive playground: https://killercoda.com/kubernetes
- CKAD exam curriculum (good scaffolding even if you don't take the exam): https://github.com/cncf/curriculum
