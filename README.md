# gitops-demo

A small GitOps demo that deploys an nginx application to Kubernetes with **Argo CD**, pulls its secret from **Keeper Security** through the **External Secrets Operator (ESO)**, and fetches the container image from a private **JFrog** registry.

Git is the single source of truth: Argo CD watches the `manifests/` directory on the `main` branch and keeps the cluster in sync with it.

## Architecture

```
                 +---------------------+
   git push ---> |  GitHub (main)      |
                 |  manifests/         |
                 +----------+----------+
                            | poll / sync
                            v
+---------------------------------------------------------+
| Kubernetes cluster                                      |
|                                                         |
|  ns: argocd   Application "gitops-demo"  (bootstrap/)   |
|                            |                            |
|                            v  applies manifests/        |
|  ns: demo                                               |
|    SecretStore --(keeper-auth)--> Keeper Security       |
|         |                                               |
|    ExternalSecret --creates--> Secret "mysql-secret"    |
|                                      |                  |
|    Deployment "nginx-demo" <---------+  (DB_PASSWORD)   |
|         ^   image pulled with "jfrog-regcred"           |
|    Service "nginx-demo" (NodePort 30080)                |
+---------------------------------------------------------+
```

## Repository layout

```
.
├── .gitignore
├── README.md
├── bootstrap/
│   └── application.yaml          # Argo CD Application (applied once, by hand)
└── manifests/                    # Everything Argo CD syncs into the "demo" namespace
    ├── 00-namespace.yaml
    ├── 01-secretstore.yaml
    ├── 02-externalsecret.yaml
    ├── 03-service.yaml
    └── 04-deployment.yaml
```

The numeric prefixes in `manifests/` only express a logical reading order (namespace → secret plumbing → networking → workload). Argo CD does not rely on file names for ordering.

## What each file means

| File | Kind | Purpose |
|------|------|---------|
| `bootstrap/application.yaml` | Argo CD `Application` | Tells Argo CD *what* to deploy, *where*, and *how* to sync. |
| `manifests/00-namespace.yaml` | `Namespace` | Creates the `demo` namespace that holds all workload resources. |
| `manifests/01-secretstore.yaml` | ESO `SecretStore` | Defines how to connect to Keeper Security. |
| `manifests/02-externalsecret.yaml` | ESO `ExternalSecret` | Declares which Keeper value becomes which Kubernetes Secret. |
| `manifests/03-service.yaml` | `Service` | Exposes nginx outside the cluster on NodePort 30080. |
| `manifests/04-deployment.yaml` | `Deployment` | Runs the nginx container. |
| `.gitignore` | – | Keeps credentials and local files out of Git. |

### `bootstrap/application.yaml`

The entry point of the GitOps loop. It lives **outside** `manifests/` on purpose: it is applied once with `kubectl` and is not managed by itself.

- `metadata.namespace: argocd` – Application objects must live in Argo CD's namespace.
- `finalizers: resources-finalizer.argocd.argoproj.io` – deleting the Application also deletes the resources it created (cascading delete).
- `source` – repository URL, branch (`targetRevision: main`) and directory (`path: manifests`) to watch.
- `destination` – the local cluster (`https://kubernetes.default.svc`) and the `demo` namespace.
- `syncPolicy.automated`
  - `prune: true` – resources removed from Git are removed from the cluster.
  - `selfHeal: true` – manual changes in the cluster (drift) are reverted to the Git state.
- `syncOptions: CreateNamespace=true` – Argo CD creates the target namespace if it is missing.
- `retry` – up to 5 attempts, starting at 5 s, doubling each time (factor 2), capped at 3 min. This helps when a dependency (such as the ESO CRDs) is not ready yet.

### `manifests/00-namespace.yaml`

Creates the `demo` namespace, labeled `app.kubernetes.io/part-of: gitops-demo`. The label is used consistently on every resource to group them as one application.

### `manifests/01-secretstore.yaml`

A `SecretStore` named `keeper-store` using the `keepersecurity` provider.

- `authRef` – points to the Kubernetes Secret `keeper-auth` (key `ksm_config`) that holds the Keeper Secrets Manager configuration. This Secret is **not** in Git (see below).
- `folderID` – the UID of the Keeper shared folder that ESO is allowed to read.

### `manifests/02-externalsecret.yaml`

An `ExternalSecret` named `mysql-secret` that uses `keeper-store`.

- `refreshInterval: 1h` – ESO re-reads Keeper every hour, so rotated values propagate automatically.
- `target.name: mysql-secret` – the Kubernetes Secret that ESO creates.
- `creationPolicy: Owner` – ESO owns the Secret and manages its lifecycle.
- `deletionPolicy: Retain` – the Secret stays even if the ExternalSecret is deleted.
- `data` – maps the Keeper record (`remoteRef.key` = record UID, `property: password`) to the Secret key `password`.

### `manifests/03-service.yaml`

A `Service` named `nginx-demo`.

- `selector: app.kubernetes.io/name: nginx-demo` – routes traffic to the Deployment's Pods.
- `type: NodePort` with `nodePort: 30080` – the app is reachable at `http://<node-ip>:30080`. The previous `ClusterIP` setting is kept as a comment.
- `port: 80` → `targetPort: http` – refers to the named container port in the Deployment.

### `manifests/04-deployment.yaml`

A `Deployment` named `nginx-demo`.

- `replicas: 1`, `revisionHistoryLimit: 3` – a single Pod; the last 3 ReplicaSets are kept for rollback.
- `imagePullSecrets: jfrog-regcred` – credentials for the private JFrog registry (not in Git).
- `image` – the nginx image hosted in JFrog; `imagePullPolicy: IfNotPresent`.
- `env: DB_PASSWORD` – read from Secret `mysql-secret`, key `password` (the Secret produced by the ExternalSecret).
- `resources` – requests 50m CPU / 64Mi memory; limits 200m CPU / 128Mi memory.
- `readinessProbe` / `livenessProbe` – HTTP GET `/` on the `http` port; readiness gates traffic, liveness restarts a stuck container.
- `nodeSelector: kubernetes.io/os: linux` – schedules only on Linux nodes.
- A `securityContext` block (no privilege escalation, drop all capabilities) is present but commented out.

### `.gitignore`

Excludes secret and credential material (`*.secret.yaml`, `*.token`, `*.key`, `*.pem`, `config.json`, `ksm-config.json`, `.env`, `ksm-config-secret.yaml`) and editor/OS files (`.vscode/`, `.idea/`, `.DS_Store`). This protects the bootstrap secrets below from being committed by accident.

## Prerequisites

- A Kubernetes cluster and `kubectl` access.
- Argo CD installed in the `argocd` namespace.
- External Secrets Operator installed (provides the `SecretStore` / `ExternalSecret` CRDs).
- A Keeper Secrets Manager application with access to the shared folder, and its config.
- A JFrog account/registry containing the nginx image.

## Secrets not stored in Git

These two Secrets must be created manually in the `demo` namespace before (or right after) the first sync:

| Secret | Used by | Content |
|--------|---------|---------|
| `keeper-auth` | `SecretStore` `keeper-store` | Key `ksm_config`: Keeper Secrets Manager configuration. |
| `jfrog-regcred` | `Deployment` `imagePullSecrets` | Docker registry credentials for JFrog. |

```bash
kubectl create namespace demo   # skip if it already exists

kubectl -n demo create secret generic keeper-auth \
  --from-file=ksm_config=./ksm-config.json

kubectl -n demo create secret docker-registry jfrog-regcred \
  --docker-server=<your-instance>.jfrog.io \
  --docker-username=<user> \
  --docker-password=<token>
```

Never commit these files; `.gitignore` already covers the common names.

## Environment-specific values to replace

When reusing this repository, update the following:

| Value | Location |
|-------|----------|
| Git repository URL | `repoURL` in `bootstrap/application.yaml` |
| Keeper shared folder UID | `folderID` in `manifests/01-secretstore.yaml` |
| Keeper record UID | `remoteRef.key` in `manifests/02-externalsecret.yaml` |
| Container image path | `image` in `manifests/04-deployment.yaml` |

## Deploy

```bash
# 1. Create the bootstrap secrets (see above)
# 2. Register the Application with Argo CD
kubectl apply -f bootstrap/application.yaml
```

From then on, Argo CD syncs `manifests/` automatically.

## Verify

```bash
kubectl -n argocd get application gitops-demo
kubectl -n demo get externalsecret,secret,deployment,pods,svc
curl http://<node-ip>:30080
```

The `ExternalSecret` should report `SecretSynced`, and the `Application` should be `Synced` / `Healthy`.

## Things to try

- **Change through Git** – edit `replicas` in `04-deployment.yaml`, commit and push; Argo CD scales the Deployment.
- **Self-heal** – run `kubectl -n demo scale deploy nginx-demo --replicas=3`; Argo CD reverts it to the value in Git.
- **Prune** – add a temporary manifest, let it sync, then delete it from Git; Argo CD removes the resource from the cluster.
- **Secret rotation** – change the password in Keeper; ESO updates `mysql-secret` within the 1 h refresh interval (Pods need a restart to pick up the new environment variable).
