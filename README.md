# External DNS — GKE + GCP Cloud DNS

This repository deploys [external-dns](https://github.com/kubernetes-sigs/external-dns) onto a GKE cluster using [helmfile](https://github.com/helmfile/helmfile). External-dns watches Kubernetes Services and automatically manages DNS records in GCP Cloud DNS, eliminating the need to manually create or update DNS entries when LoadBalancer IPs are assigned or rotated.

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Architecture](#architecture)
3. [Authentication — Workload Identity](#authentication--workload-identity)
4. [Repository Structure](#repository-structure)
5. [Configuration Reference](#configuration-reference)
6. [GCP Prerequisites](#gcp-prerequisites)
7. [Deployment](#deployment)
8. [Using External DNS — Annotating Services](#using-external-dns--annotating-services)
9. [DNS Record Lifecycle](#dns-record-lifecycle)
10. [Ownership and TXT Records](#ownership-and-txt-records)
11. [Verification and Troubleshooting](#verification-and-troubleshooting)

---

## How It Works

External-dns is a controller that runs inside your Kubernetes cluster. On every reconciliation interval (default: 1 minute) it:

1. **Queries the Kubernetes API** for all Services of type `LoadBalancer` across all namespaces.
2. **Reads the `external-dns.alpha.kubernetes.io/hostname` annotation** on each Service. This annotation tells external-dns what DNS name to create for that Service.
3. **Reads the Service's `status.loadBalancer.ingress[].ip`** — the external IP assigned by GKE/GCP after the cloud load balancer is provisioned.
4. **Calls the GCP Cloud DNS API** to upsert an `A` record mapping that hostname to the IP, and a companion `TXT` record that records ownership metadata.
5. **Deletes stale records** — if a Service is deleted or its annotation is removed, external-dns removes the `A` and `TXT` records it previously created (because `policy: sync` is set).

No manual `gcloud dns record-sets create` commands are needed. The moment GKE assigns an IP to a LoadBalancer Service that carries the annotation, the DNS record appears automatically.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  GKE Cluster                                                    │
│                                                                 │
│  ┌──────────────────────┐      ┌──────────────────────────────┐ │
│  │  namespace: default  │      │  namespace: external-dns     │ │
│  │                      │      │                              │ │
│  │  Service (LB)        │      │  Pod: external-dns           │ │
│  │  ┌────────────────┐  │      │  ┌──────────────────────┐   │ │
│  │  │ type:          │  │      │  │  1. Watch all         │   │ │
│  │  │  LoadBalancer  │  │      │  │     Services          │   │ │
│  │  │                │  │      │  │  2. Read annotations  │   │ │
│  │  │ annotation:    │◄─┼──────┼──│  3. Read LB IPs       │   │ │
│  │  │  hostname:     │  │      │  │  4. Call Cloud DNS    │   │ │
│  │  │  foo.example   │  │      │  │     API               │   │ │
│  │  │                │  │      │  └──────────┬───────────┘   │ │
│  │  │ status:        │  │      │             │ Workload       │ │
│  │  │  LB IP: 1.2.3.4│  │      │             │ Identity       │ │
│  │  └────────────────┘  │      └─────────────┼───────────────┘ │
│  └──────────────────────┘                    │                  │
└─────────────────────────────────────────────┼──────────────────┘
                                              │ GCP IAM (federated)
                                              ▼
                              ┌───────────────────────────┐
                              │  GCP Cloud DNS            │
                              │                           │
                              │  Zone: safabayar.net      │
                              │                           │
                              │  foo.safabayar.net  A     │
                              │    → 1.2.3.4              │
                              │                           │
                              │  foo.safabayar.net  TXT   │
                              │    → "heritage=..."       │
                              └───────────────────────────┘
```

External-dns communicates with Cloud DNS using the GCP IAM Service Account identity federated through GKE Workload Identity — no credentials are stored in the cluster.

---

## Authentication — Workload Identity

Workload Identity is the GKE-native way to grant a Kubernetes workload access to GCP APIs without embedding service account keys as Kubernetes Secrets. It works through identity federation:

```
Pod (runs as K8s SA "external-dns")
  │
  │  GKE injects federated token via the metadata server
  ▼
GKE Metadata Server
  │
  │  Token is accepted by Google's STS (Security Token Service)
  ▼
GCP IAM
  │
  │  K8s SA "external-dns/external-dns" is bound to
  │  GCP SA "external-dns@PROJECT.iam.gserviceaccount.com"
  ▼
GCP Cloud DNS API  ← called with GCP SA identity (roles/dns.admin)
```

**The binding is two-sided:**

| Side | What it does | How it's set |
|---|---|---|
| GCP IAM binding | Allows the K8s SA to impersonate the GCP SA | `gcloud iam service-accounts add-iam-policy-binding` with `roles/iam.workloadIdentityUser` |
| K8s SA annotation | Tells GKE which GCP SA to federate to | `iam.gke.io/gcp-service-account` annotation on the K8s ServiceAccount |

The annotation on the Kubernetes ServiceAccount is injected by the helmfile `set` block:

```yaml
# helmfile.yaml
set:
  - name: serviceAccount.annotations.iam\.gke\.io/gcp-service-account
    value: external-dns@{{ requiredEnv "GCP_PROJECT_ID" }}.iam.gserviceaccount.com
```

The escaped dots (`\.`) are required because Helm's `set` uses dots as path separators; the annotation key contains literal dots that must not be treated as separators.

---

## Repository Structure

```
external-dns-k8s/
├── helmfile.yaml          # Chart source, version, release config, env var injection
├── values/
│   └── external-dns.yaml  # external-dns chart values (provider, sources, policy, etc.)
└── .env.example           # Documents required environment variables
```

### `helmfile.yaml`

```yaml
repositories:
  - name: external-dns
    url: https://kubernetes-sigs.github.io/external-dns/

releases:
  - name: external-dns
    namespace: external-dns
    createNamespace: true          # helmfile creates the namespace if absent
    chart: external-dns/external-dns
    version: "1.15.0"
    values:
      - values/external-dns.yaml   # static values file
    set:
      # These two values contain the GCP project ID, sourced from the environment.
      # requiredEnv fails fast if the variable is unset — prevents a silent misconfiguration.
      - name: google.project
        value: {{ requiredEnv "GCP_PROJECT_ID" }}
      - name: serviceAccount.annotations.iam\.gke\.io/gcp-service-account
        value: external-dns@{{ requiredEnv "GCP_PROJECT_ID" }}.iam.gserviceaccount.com
```

The `set` block overrides specific chart values at deploy time using the `GCP_PROJECT_ID` environment variable. This keeps the project ID out of version-controlled files.

### `values/external-dns.yaml`

```yaml
provider:
  name: google                # Use GCP Cloud DNS driver

sources:
  - service                   # Watch Services (LoadBalancer type is relevant)

domainFilters:
  - safabayar.net             # Only manage records under this zone

policy: sync                  # Upsert + delete (use "upsert-only" to disable deletes)

txtOwnerId: external-dns-gke  # Stamped on TXT records to identify this instance

interval: 1m                  # Reconciliation frequency

logLevel: info
logFormat: json

rbac:
  create: true                # ClusterRole + ClusterRoleBinding for all-namespace watch

serviceAccount:
  create: true
  name: external-dns
  annotations: {}             # Workload Identity annotation injected by helmfile

resources:
  requests: { cpu: 50m, memory: 64Mi }
  limits:   { cpu: 100m, memory: 128Mi }
```

---

## Configuration Reference

| Value | Default | Description |
|---|---|---|
| `provider.name` | `google` | DNS provider. Must be `google` for GCP Cloud DNS. |
| `sources` | `[service]` | Kubernetes resource types to watch for hostname annotations. |
| `domainFilters` | `[safabayar.net]` | Restricts external-dns to only manage records under these zones. Records outside this list are never touched. |
| `policy` | `sync` | `sync`: create and delete. `upsert-only`: create but never delete. |
| `txtOwnerId` | `external-dns-gke` | Unique string stamped into TXT records. Used to determine ownership. Must be unique per cluster if you run multiple external-dns instances. |
| `interval` | `1m` | How often external-dns reconciles Kubernetes state against Cloud DNS. |
| `google.project` | *(set by helmfile)* | GCP project ID where the Cloud DNS zone lives. |
| `rbac.create` | `true` | Creates the ClusterRole and ClusterRoleBinding allowing external-dns to list/watch Services across all namespaces. |

---

## GCP Prerequisites

Run these commands once before deploying. Replace the variable values with your own.

```bash
export PROJECT_ID="your-gcp-project-id"
export DNS_ZONE="safabayar-net"        # Cloud DNS managed zone name (not the domain)
export GKE_CLUSTER="your-cluster"
export GKE_REGION="us-central1"
```

### 1. Enable GCP APIs

```bash
gcloud services enable dns.googleapis.com container.googleapis.com \
  --project=$PROJECT_ID
```

### 2. Create the GCP IAM Service Account

```bash
gcloud iam service-accounts create external-dns \
  --display-name="External DNS" \
  --project=$PROJECT_ID
```

### 3. Grant DNS Admin Role

`roles/dns.admin` allows the service account to create, update, and delete records in any Cloud DNS zone in the project. If you need tighter scoping, you can grant this at the zone level instead of the project level.

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:external-dns@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/dns.admin"
```

### 4. Enable Workload Identity on the GKE Cluster

If your cluster was not created with Workload Identity enabled, update it. This restarts node pools — plan for a brief disruption.

```bash
gcloud container clusters update $GKE_CLUSTER \
  --region=$GKE_REGION \
  --workload-pool=$PROJECT_ID.svc.id.goog \
  --project=$PROJECT_ID
```

To verify Workload Identity is active:

```bash
gcloud container clusters describe $GKE_CLUSTER \
  --region=$GKE_REGION \
  --project=$PROJECT_ID \
  --format="value(workloadIdentityConfig.workloadPool)"
# Expected output: your-project.svc.id.goog
```

### 5. Bind the Kubernetes Service Account to the GCP Service Account

This IAM binding is the bridge between the cluster identity and the GCP identity. The member format is `serviceAccount:PROJECT.svc.id.goog[NAMESPACE/KSA_NAME]` where `external-dns/external-dns` is `namespace/kubernetes-service-account-name`.

```bash
gcloud iam service-accounts add-iam-policy-binding \
  external-dns@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:$PROJECT_ID.svc.id.goog[external-dns/external-dns]" \
  --project=$PROJECT_ID
```

### 6. Verify Your Cloud DNS Zone Exists

External-dns does not create the managed zone — it only manages records inside an existing zone. If you don't have one:

```bash
gcloud dns managed-zones create $DNS_ZONE \
  --description="Public zone for safabayar.net" \
  --dns-name="safabayar.net." \
  --visibility=public \
  --project=$PROJECT_ID
```

---

## Deployment

### Prerequisites

- `helmfile` installed locally
- `helm` installed locally
- `kubectl` configured against your GKE cluster
- `GCP_PROJECT_ID` environment variable set

```bash
# Install helmfile (if needed)
brew install helmfile        # macOS
# or: https://github.com/helmfile/helmfile/releases

# Install helm-diff plugin (required by helmfile)
helm plugin install https://github.com/databus23/helm-diff
```

### Deploy

```bash
cd /path/to/external-dns-k8s

export GCP_PROJECT_ID="your-gcp-project-id"

# Preview what will be applied (dry-run)
helmfile diff

# Deploy
helmfile sync
```

`helmfile sync` adds the Helm repository, pulls the chart, renders the templates with your values, and installs or upgrades the release into the `external-dns` namespace.

### Upgrade

To upgrade to a new chart version, change `version` in `helmfile.yaml` and run:

```bash
helmfile sync
```

To change a value without editing files:

```bash
helmfile sync --set interval=30s
```

### Teardown

```bash
helmfile destroy
```

This uninstalls the Helm release but does not delete DNS records that were already created. External-dns only deletes records on its next sync cycle if it is still running and the source Service no longer exists. After `helmfile destroy`, orphaned records must be removed manually via `gcloud dns record-sets delete`.

---

## Using External DNS — Annotating Services

External-dns takes no action on a Service unless it carries the hostname annotation. This is intentional — it means you opt in per-Service.

### Annotation

```
external-dns.alpha.kubernetes.io/hostname: <desired-fqdn>
```

Multiple hostnames are comma-separated:

```
external-dns.alpha.kubernetes.io/hostname: gateway.safabayar.net,api.safabayar.net
```

### Example: Gateway Service

To expose the gateway's LoadBalancer with a DNS record, add the annotation via the gateway Helm chart's values:

```yaml
# In gateway/values-override.yaml
service:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: gateway.safabayar.net
```

Then upgrade the gateway release:

```bash
helm upgrade gateway ./helm/gateway -f values-override.yaml
```

Or inline:

```bash
helm upgrade gateway ./helm/gateway \
  --set 'service.annotations.external-dns\.alpha\.kubernetes\.io/hostname=gateway.safabayar.net'
```

After GKE assigns the LoadBalancer IP and external-dns runs its next sync (~1 minute), the following records appear in Cloud DNS:

```
gateway.safabayar.net.  300  IN  A    34.x.x.x
gateway.safabayar.net.  300  IN  TXT  "heritage=external-dns,external-dns/owner=external-dns-gke,..."
```

### TTL Override (optional)

You can override the default TTL (300 seconds) per-Service:

```
external-dns.alpha.kubernetes.io/ttl: "60"
```

---

## DNS Record Lifecycle

```
Service created with annotation
        │
        ▼
GKE provisions Cloud Load Balancer
        │
        ▼
Service status.loadBalancer.ingress[0].ip is populated
        │
        ▼
external-dns reconciliation fires (up to 1 minute later)
        │
        ├── Calls Cloud DNS: upsert A record  (hostname → IP)
        └── Calls Cloud DNS: upsert TXT record (ownership marker)

Service annotation removed or Service deleted
        │
        ▼
external-dns reconciliation fires
        │
        ├── Reads TXT record — ownership matches txtOwnerId
        ├── Calls Cloud DNS: delete A record
        └── Calls Cloud DNS: delete TXT record
```

If `policy: upsert-only` is used, the delete steps are skipped — records persist after the Service is gone and must be cleaned up manually.

---

## Ownership and TXT Records

Every DNS record created by external-dns is accompanied by a `TXT` record:

```
gateway.safabayar.net.  300  IN  TXT  "heritage=external-dns,external-dns/owner=external-dns-gke,external-dns/resource=service/default/gateway"
```

The `txtOwnerId` value (`external-dns-gke`) is embedded in this record. External-dns will only modify or delete records whose TXT owner matches its own configured `txtOwnerId`. This prevents one external-dns instance from interfering with records managed by another instance (e.g., in a different cluster or namespace).

**If you run a second cluster**, give it a different `txtOwnerId` (e.g., `external-dns-gke-staging`) to keep ownership separate.

---

## Verification and Troubleshooting

### Check the pod is running

```bash
kubectl get pods -n external-dns
# NAME                            READY   STATUS    RESTARTS   AGE
# external-dns-6d9b7c5f8d-xk9vp   1/1     Running   0          2m
```

### Watch logs

```bash
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns -f
```

A successful sync looks like:

```json
{"level":"info","msg":"Desired change: CREATE gateway.safabayar.net A [34.x.x.x]"}
{"level":"info","msg":"Desired change: CREATE gateway.safabayar.net TXT [...]"}
{"level":"info","msg":"2 record(s) in zone safabayar.net. were successfully updated"}
```

If no Services have the annotation yet:

```json
{"level":"info","msg":"All records are already up to date"}
```

### Verify the Workload Identity annotation on the K8s ServiceAccount

```bash
kubectl get serviceaccount -n external-dns external-dns -o jsonpath='{.metadata.annotations}'
# {"iam.gke.io/gcp-service-account":"external-dns@your-project.iam.gserviceaccount.com"}
```

If the annotation is missing, helmfile did not apply the `set` block — check that `GCP_PROJECT_ID` was exported before running `helmfile sync`.

### Verify records in Cloud DNS

```bash
gcloud dns record-sets list \
  --zone=$DNS_ZONE \
  --project=$PROJECT_ID
```

### Common issues

| Symptom | Likely cause | Fix |
|---|---|---|
| Pod crashes with `Permission denied` | Workload Identity IAM binding missing or GCP SA lacks `dns.admin` | Re-run steps 3 and 5 of GCP prerequisites |
| Logs show `No changes` but record is missing | Service has no annotation | Add `external-dns.alpha.kubernetes.io/hostname` annotation |
| Record not deleted after Service removed | `policy: upsert-only` is set | Change to `policy: sync`, or delete the record manually |
| Pod shows `Error: required env var GCP_PROJECT_ID is not set` | Environment variable not exported | `export GCP_PROJECT_ID=...` before `helmfile sync` |
| Record created but points to wrong IP | Multiple replicas of gateway or IP changed | external-dns will self-correct on next sync cycle |
| TXT ownership conflict | Two clusters sharing the same `txtOwnerId` | Set a unique `txtOwnerId` per cluster |
