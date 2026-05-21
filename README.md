# External DNS — GKE + GCP Cloud DNS

This repository deploys [external-dns](https://github.com/kubernetes-sigs/external-dns) using [helmfile](https://github.com/helmfile/helmfile) with two supported target environments:

| Environment | Cluster | LoadBalancer IPs | Authentication |
|---|---|---|---|
| `gke` | Google Kubernetes Engine | GCP Cloud Load Balancer | Workload Identity (no secrets) |
| `kind` | kind (local, on-prem) | MetalLB (L2 mode) | GCP Service Account JSON key |

For a dedicated step-by-step kind guide see [KIND.md](KIND.md).

Both environments write DNS records to the same GCP Cloud DNS zone. The `kind` environment is intended for local development and on-premises setups where GKE-specific features like Workload Identity are unavailable.

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Architecture](#architecture)
   - [GKE Architecture](#gke-architecture)
   - [kind Architecture](#kind-architecture)
3. [Key Differences Between Environments](#key-differences-between-environments)
4. [Repository Structure](#repository-structure)
5. [Configuration Reference](#configuration-reference)
6. [GKE Installation](#gke-installation)
   - [GCP Prerequisites](#gcp-prerequisites)
   - [Authentication — Workload Identity](#authentication--workload-identity)
   - [Deploy on GKE](#deploy-on-gke)
7. [kind On-Premises Installation](#kind-on-premises-installation)
   - [How kind Differs](#how-kind-differs)
   - [kind Prerequisites](#kind-prerequisites)
   - [Authentication — Service Account Key](#authentication--service-account-key)
   - [Create the kind Cluster](#create-the-kind-cluster)
   - [Configure MetalLB](#configure-metallb)
   - [Create the Credentials Secret](#create-the-credentials-secret)
   - [Deploy on kind](#deploy-on-kind)
8. [Using External DNS — Annotating Services](#using-external-dns--annotating-services)
9. [DNS Record Lifecycle](#dns-record-lifecycle)
10. [Ownership and TXT Records](#ownership-and-txt-records)
11. [Verification and Troubleshooting](#verification-and-troubleshooting)

---

## How It Works

External-dns is a controller that runs inside your Kubernetes cluster. On every reconciliation interval (default: 1 minute) it:

1. **Queries the Kubernetes API** for all Services of type `LoadBalancer` across all namespaces.
2. **Reads the `external-dns.alpha.kubernetes.io/hostname` annotation** on each Service. This annotation tells external-dns what DNS name to create for that Service.
3. **Reads the Service's `status.loadBalancer.ingress[].ip`** — the external IP assigned to the Service by the load balancer controller (GCP Cloud Load Balancer on GKE, MetalLB on kind).
4. **Calls the GCP Cloud DNS API** to upsert an `A` record mapping that hostname to the IP, and a companion `TXT` record that records ownership metadata.
5. **Deletes stale records** — if a Service is deleted or its annotation is removed, external-dns removes the `A` and `TXT` records it previously created (because `policy: sync` is set).

No manual `gcloud dns record-sets create` commands are needed. The moment the load balancer controller assigns an IP to an annotated Service, the DNS record appears automatically.

---

## Architecture

### GKE Architecture

On GKE, the cloud controller manager assigns real public IP addresses to LoadBalancer Services, and Workload Identity federates the pod's Kubernetes service account identity to a GCP IAM service account — no credentials are stored in the cluster.

```
┌──────────────────────────────────────────────────────────────────┐
│  GKE Cluster                                                     │
│                                                                  │
│  ┌───────────────────────┐    ┌───────────────────────────────┐  │
│  │  namespace: default   │    │  namespace: external-dns      │  │
│  │                       │    │                               │  │
│  │  Service (LB)         │    │  Pod: external-dns            │  │
│  │  ┌─────────────────┐  │    │  ┌─────────────────────────┐  │  │
│  │  │ type:           │  │    │  │ 1. Watch all Services   │  │  │
│  │  │  LoadBalancer   │◄─┼────┼──│ 2. Read annotations     │  │  │
│  │  │                 │  │    │  │ 3. Read LB IPs          │  │  │
│  │  │ annotation:     │  │    │  │ 4. Call Cloud DNS API   │  │  │
│  │  │  hostname: ...  │  │    │  └──────────┬──────────────┘  │  │
│  │  │                 │  │    │             │ Workload         │  │
│  │  │ status:         │  │    │             │ Identity         │  │
│  │  │  LB IP: 34.x.x.x│  │    └─────────────┼─────────────────┘  │
│  │  └─────────────────┘  │                  │                     │
│  └───────────────────────┘                  │ GCP IAM (federated) │
└─────────────────────────────────────────────┼─────────────────────┘
                                              │
                                              ▼
                              ┌───────────────────────────┐
                              │  GCP Cloud DNS            │
                              │  Zone: safabayar.net      │
                              │                           │
                              │  gateway.safabayar.net A  │
                              │    → 34.x.x.x             │
                              └───────────────────────────┘
```

### kind Architecture

On kind, there is no cloud controller, so LoadBalancer Services would normally stay in `<pending>` state forever. MetalLB solves this by watching for LoadBalancer Services and assigning IPs from a configured pool carved out of the Docker bridge network that kind uses. Those IPs are routable on the host machine (and between kind nodes) because MetalLB uses L2/ARP to advertise them. External-dns then reads those IPs exactly as it does on GKE.

Authentication uses a GCP Service Account JSON key stored as a Kubernetes Secret because the GKE metadata server that powers Workload Identity does not exist in a kind cluster.

```
┌──────────────────────────────────────────────────────────────────┐
│  kind Cluster (Docker network: 172.18.0.0/16)                   │
│                                                                  │
│  ┌───────────────────────┐    ┌───────────────────────────────┐  │
│  │  namespace: default   │    │  namespace: external-dns      │  │
│  │                       │    │                               │  │
│  │  Service (LB)         │    │  Pod: external-dns            │  │
│  │  ┌─────────────────┐  │    │  ┌─────────────────────────┐  │  │
│  │  │ type:           │  │    │  │ 1. Watch all Services   │  │  │
│  │  │  LoadBalancer   │◄─┼────┼──│ 2. Read annotations     │  │  │
│  │  │                 │  │    │  │ 3. Read LB IPs          │  │  │
│  │  │ annotation:     │  │    │  │ 4. Call Cloud DNS API   │  │  │
│  │  │  hostname: ...  │  │    │  └──────────┬──────────────┘  │  │
│  │  │                 │  │    │             │ SA JSON key      │  │
│  │  │ status:         │  │    │             │ from Secret      │  │
│  │  │  LB IP: 172.18.x│◄─┼────┼─────────────┼─────────────────┘  │
│  │  └─────────────────┘  │    │  ┌──────────┴──────────────┐  │  │
│  └───────────────────────┘    │  │  namespace: metallb-sys  │  │  │
│                               │  │  MetalLB controller      │  │  │
│                               │  │  assigns 172.18.255.x IP │  │  │
│                               │  │  via ARP on Docker net   │  │  │
│                               │  └─────────────────────────┘  │  │
│                               └───────────────────────────────┘  │
└─────────────────────────────────────────────┬────────────────────┘
                                              │ HTTPS (GCP API)
                                              ▼
                              ┌───────────────────────────┐
                              │  GCP Cloud DNS            │
                              │  Zone: safabayar.net      │
                              │                           │
                              │  gateway.safabayar.net A  │
                              │    → 172.18.255.200        │
                              └───────────────────────────┘
```

Note: on kind the DNS `A` record points to a private RFC1918 address. This is useful for local testing of the full DNS automation workflow even without a publicly routable IP.

---

## Key Differences Between Environments

| Concern | GKE (`-e gke`) | kind (`-e kind`) |
|---|---|---|
| **LoadBalancer IPs** | Assigned by GCP Cloud Load Balancer controller | Assigned by MetalLB from a Docker network IP pool |
| **Authentication to Cloud DNS** | Workload Identity — no secrets in cluster | GCP Service Account JSON key stored as a K8s Secret |
| **K8s ServiceAccount annotation** | `iam.gke.io/gcp-service-account: external-dns@PROJECT.iam.gserviceaccount.com` | None |
| **MetalLB** | Not installed (`metallb.enabled: false`) | Installed (`metallb.enabled: true`) |
| **TXT owner ID** | `external-dns-gke` | `external-dns-kind` |
| **IP addresses in DNS** | Public IPs (routable from internet) | Private Docker IPs (routable from host only) |

---

## Repository Structure

```
external-dns-k8s/
├── helmfile.yaml                    # Repositories, environments, releases
├── environments/
│   ├── gke.yaml                     # GKE env vars (metallb.enabled: false)
│   └── kind.yaml                    # kind env vars (metallb.enabled: true)
├── values/
│   ├── external-dns.yaml            # Common chart values (provider, sources, policy)
│   ├── external-dns-gke.yaml        # GKE overrides (txtOwnerId, SA annotation placeholder)
│   └── external-dns-kind.yaml       # kind overrides (txtOwnerId, SA key secret ref)
├── kind-cluster.yaml                # kind cluster definition
├── metallb-config.yaml              # MetalLB IPAddressPool + L2Advertisement
└── .env.example                     # Documents required environment variables
```

### `helmfile.yaml`

```yaml
repositories:
  - name: external-dns
    url: https://kubernetes-sigs.github.io/external-dns/
  - name: metallb
    url: https://metallb.github.io/metallb

environments:
  gke:
    values:
      - environments/gke.yaml
  kind:
    values:
      - environments/kind.yaml

releases:
  - name: metallb
    namespace: metallb-system
    createNamespace: true
    chart: metallb/metallb
    version: "0.14.9"
    installed: {{ .Values | get "metallb.enabled" false }}  # skipped on GKE

  - name: external-dns
    namespace: external-dns
    createNamespace: true
    chart: external-dns/external-dns
    version: "1.15.0"
    values:
      - values/external-dns.yaml
      - values/external-dns-{{ .Environment.Name }}.yaml   # env-specific overlay
    set:
      - name: google.project
        value: {{ requiredEnv "GCP_PROJECT_ID" }}
      {{- if eq .Environment.Name "gke" }}
      - name: serviceAccount.annotations.iam\.gke\.io/gcp-service-account
        value: external-dns@{{ requiredEnv "GCP_PROJECT_ID" }}.iam.gserviceaccount.com
      {{- end }}
```

The `set` block injects the GCP project ID from the `GCP_PROJECT_ID` environment variable. On GKE it also injects the Workload Identity annotation. On kind that block is omitted and the chart instead reads credentials from the Secret referenced in `values/external-dns-kind.yaml`.

`requiredEnv` causes helmfile to exit immediately with a clear error if the variable is not set, preventing a silent misconfiguration.

### `values/external-dns.yaml` (common)

Values shared by both environments. Environment-specific files overlay on top.

```yaml
provider:
  name: google

sources:
  - service

domainFilters:
  - safabayar.net

policy: sync

interval: 1m

logLevel: info
logFormat: json

rbac:
  create: true

serviceAccount:
  create: true
  name: external-dns

resources:
  requests: { cpu: 50m, memory: 64Mi }
  limits:   { cpu: 100m, memory: 128Mi }
```

### `values/external-dns-gke.yaml`

```yaml
txtOwnerId: external-dns-gke

serviceAccount:
  annotations: {}   # Workload Identity annotation injected by helmfile set block
```

### `values/external-dns-kind.yaml`

```yaml
txtOwnerId: external-dns-kind

google:
  serviceAccountSecret: external-dns-gcp-credentials
  serviceAccountSecretKey: credentials.json

serviceAccount:
  annotations: {}
```

---

## Configuration Reference

| Value | GKE default | kind default | Description |
|---|---|---|---|
| `provider.name` | `google` | `google` | DNS provider. Both use GCP Cloud DNS. |
| `sources` | `[service]` | `[service]` | Kubernetes resources to watch. |
| `domainFilters` | `[safabayar.net]` | `[safabayar.net]` | Only manage records under these zones. |
| `policy` | `sync` | `sync` | `sync`: upsert + delete. `upsert-only`: never delete. |
| `txtOwnerId` | `external-dns-gke` | `external-dns-kind` | Stamped into TXT records to identify the owner instance. Different per environment to prevent conflicts. |
| `interval` | `1m` | `1m` | Reconciliation frequency. |
| `google.project` | *(from env var)* | *(from env var)* | GCP project ID where Cloud DNS zone lives. |
| `google.serviceAccountSecret` | *(not set)* | `external-dns-gcp-credentials` | K8s Secret name containing `credentials.json`. kind only. |
| `rbac.create` | `true` | `true` | Creates ClusterRole + ClusterRoleBinding for all-namespace watch. |

---

## GKE Installation

### GCP Prerequisites

Run these commands once. All subsequent deploys only need `helmfile sync -e gke`.

```bash
export PROJECT_ID="your-gcp-project-id"
export DNS_ZONE="safabayar-net"      # Cloud DNS managed zone name
export GKE_CLUSTER="your-cluster"
export GKE_REGION="us-central1"

# 1. Enable required APIs
gcloud services enable dns.googleapis.com container.googleapis.com \
  --project=$PROJECT_ID

# 2. Create GCP service account
gcloud iam service-accounts create external-dns \
  --display-name="External DNS" \
  --project=$PROJECT_ID

# 3. Grant DNS admin role
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:external-dns@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/dns.admin"

# 4. Enable Workload Identity on the cluster (restarts node pools)
gcloud container clusters update $GKE_CLUSTER \
  --region=$GKE_REGION \
  --workload-pool=$PROJECT_ID.svc.id.goog \
  --project=$PROJECT_ID

# 5. Bind the K8s service account to the GCP service account
#    Format: serviceAccount:PROJECT.svc.id.goog[K8S_NAMESPACE/K8S_SA_NAME]
gcloud iam service-accounts add-iam-policy-binding \
  external-dns@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:$PROJECT_ID.svc.id.goog[external-dns/external-dns]" \
  --project=$PROJECT_ID

# 6. Verify your Cloud DNS zone exists (create if not)
gcloud dns managed-zones describe $DNS_ZONE --project=$PROJECT_ID \
  || gcloud dns managed-zones create $DNS_ZONE \
       --description="Public zone for safabayar.net" \
       --dns-name="safabayar.net." \
       --visibility=public \
       --project=$PROJECT_ID
```

### Authentication — Workload Identity

Workload Identity federates the Kubernetes pod's service account identity to a GCP IAM identity without storing any credentials in the cluster. The binding is two-sided:

```
Pod (K8s SA: external-dns/external-dns)
  │
  │  GKE metadata server provides a federated OIDC token
  ▼
Google STS (Security Token Service)
  │
  │  Token validated against IAM binding:
  │  roles/iam.workloadIdentityUser on external-dns@PROJECT.iam.gserviceaccount.com
  │  granted to serviceAccount:PROJECT.svc.id.goog[external-dns/external-dns]
  ▼
GCP IAM (acting as external-dns@PROJECT.iam.gserviceaccount.com)
  │
  │  roles/dns.admin
  ▼
GCP Cloud DNS API
```

The K8s ServiceAccount must carry the annotation `iam.gke.io/gcp-service-account`. This is injected by the helmfile `set` block rather than hardcoded in a values file, so the project ID stays out of version control. The dots in the annotation key are escaped (`\.`) because Helm uses dots as path separators in `--set` syntax.

### Deploy on GKE

```bash
cd /path/to/external-dns-k8s

export GCP_PROJECT_ID="your-gcp-project-id"

# Preview changes
helmfile diff -e gke

# Deploy
helmfile sync -e gke
```

---

## kind On-Premises Installation

### How kind Differs

Running external-dns on kind requires solving two problems that GKE handles transparently:

**Problem 1 — No LoadBalancer IP assignment.**
In a standard Kubernetes cluster without a cloud provider (like kind), Services of type `LoadBalancer` are created but never get an external IP — they sit in `<pending>` indefinitely. External-dns cannot create a DNS record without an IP to point to.

**Solution: MetalLB.** MetalLB is a software load balancer that runs inside the cluster. It watches for `LoadBalancer` Services and assigns IP addresses from a pre-configured pool. In L2 mode it uses ARP to advertise those IPs on the local network — in kind's case, on the Docker bridge network — making them reachable from the host machine and between nodes.

**Problem 2 — No Workload Identity.**
Workload Identity relies on the GKE node metadata server, which proxies requests to Google's STS. That infrastructure does not exist in kind. External-dns therefore cannot automatically obtain GCP credentials from the environment.

**Solution: Service Account JSON key Secret.** A GCP service account key is downloaded and stored as a Kubernetes Secret. External-dns mounts that Secret and uses it to authenticate to the Cloud DNS API directly.

### kind Prerequisites

```bash
# kind
brew install kind           # macOS
# https://kind.sigs.k8s.io/docs/user/quick-start/#installation

# kubectl
brew install kubectl

# helmfile + helm
brew install helmfile
helm plugin install https://github.com/databus23/helm-diff

# Docker (must be running — kind runs clusters as Docker containers)
docker info
```

### Authentication — Service Account Key

The same GCP service account used for GKE (`external-dns@PROJECT.iam.gserviceaccount.com`) can be reused. The only change is the authentication method: instead of Workload Identity, a JSON key is downloaded and stored in the cluster.

```
Pod (K8s SA: external-dns)
  │
  │  Mounts Secret "external-dns-gcp-credentials"
  │  Key: credentials.json
  ▼
GCP Cloud DNS API client library
  │
  │  Reads GOOGLE_APPLICATION_CREDENTIALS or key file directly
  ▼
GCP IAM (acting as external-dns@PROJECT.iam.gserviceaccount.com)
  │
  │  roles/dns.admin
  ▼
GCP Cloud DNS API
```

The Secret is created manually before deploying because it contains sensitive key material that must not be stored in the git repository.

**Create the GCP service account and key** (skip if already done for GKE):

```bash
export PROJECT_ID="your-gcp-project-id"

# Create service account if it doesn't exist
gcloud iam service-accounts create external-dns \
  --display-name="External DNS" \
  --project=$PROJECT_ID

# Grant DNS admin
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:external-dns@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/dns.admin"

# Download a JSON key
gcloud iam service-accounts keys create external-dns-sa-key.json \
  --iam-account=external-dns@$PROJECT_ID.iam.gserviceaccount.com \
  --project=$PROJECT_ID
```

Keep `external-dns-sa-key.json` out of version control. Add it to `.gitignore`:

```bash
echo "*.json" >> .gitignore
```

### Create the kind Cluster

`kind-cluster.yaml` defines a single control-plane node and one worker. MetalLB operates on the Docker bridge network kind creates automatically — no special CNI configuration is needed.

```bash
kind create cluster --config kind-cluster.yaml --name external-dns-dev

# Verify
kubectl cluster-info --context kind-external-dns-dev
kubectl get nodes
```

### Configure MetalLB

MetalLB needs to know which IP range to use. The range must fall within the Docker network subnet that kind created.

**Step 1 — Find the Docker subnet:**

```bash
docker network inspect kind | grep -A2 '"Subnet"'
# Example output:
#   "Subnet": "172.18.0.0/16",
#   "Gateway": "172.18.0.1"
```

**Step 2 — Edit `metallb-config.yaml` if your subnet differs:**

The file defaults to `172.18.255.200-172.18.255.250`. If your kind Docker network uses a different subnet (e.g. `172.19.0.0/16`), change the pool accordingly:

```yaml
spec:
  addresses:
    - 172.19.255.200-172.19.255.250
```

Kind node IPs use the low end of the range (e.g. `172.18.0.2`, `172.18.0.3`). Using the upper end (`172.18.255.x`) avoids collisions.

**Step 3 — Deploy MetalLB via helmfile, then apply the config:**

```bash
export GCP_PROJECT_ID="your-gcp-project-id"

# Install MetalLB and external-dns
helmfile sync -e kind

# Wait for MetalLB pods to be Running before applying the config
kubectl rollout status deployment -n metallb-system controller
kubectl rollout status daemonset  -n metallb-system speaker

# Apply the IP pool and L2 advertisement
kubectl apply -f metallb-config.yaml
```

The MetalLB config must be applied after the MetalLB CRDs are installed by the chart. Running `kubectl apply -f metallb-config.yaml` before MetalLB is ready will fail because the `IPAddressPool` and `L2Advertisement` CRDs do not exist yet.

**Verify MetalLB is assigning IPs:**

```bash
# Create a test LoadBalancer service
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80 --type=LoadBalancer

# Should show an EXTERNAL-IP from the MetalLB pool (not <pending>)
kubectl get svc nginx
# NAME    TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
# nginx   LoadBalancer   10.96.x.x     172.18.255.200   80:3xxxx/TCP   30s

# Clean up
kubectl delete pod nginx
kubectl delete svc nginx
```

### Create the Credentials Secret

The Secret must exist in the `external-dns` namespace before external-dns starts. Create the namespace and Secret before or immediately after running `helmfile sync -e kind`.

```bash
kubectl create namespace external-dns

kubectl create secret generic external-dns-gcp-credentials \
  --namespace external-dns \
  --from-file=credentials.json=./external-dns-sa-key.json
```

Verify the Secret:

```bash
kubectl get secret -n external-dns external-dns-gcp-credentials
# NAME                            TYPE     DATA   AGE
# external-dns-gcp-credentials    Opaque   1      5s
```

External-dns mounts this Secret at startup. If the Secret is missing, the pod fails with `credentials.json: no such file or directory`. The pod does not need to be restarted after the Secret is created — if the Secret was missing at pod start, delete the pod and let it restart once the Secret exists.

### Deploy on kind

```bash
cd /path/to/external-dns-k8s

export GCP_PROJECT_ID="your-gcp-project-id"

# Preview
helmfile diff -e kind

# Deploy MetalLB + external-dns
helmfile sync -e kind

# Apply MetalLB IP pool config (after MetalLB pods are Ready)
kubectl rollout status deployment -n metallb-system controller
kubectl apply -f metallb-config.yaml
```

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

After the load balancer controller assigns an IP (GCP or MetalLB) and external-dns runs its next sync (~1 minute), the following records appear in Cloud DNS:

```
gateway.safabayar.net.  300  IN  A    <IP>
gateway.safabayar.net.  300  IN  TXT  "heritage=external-dns,external-dns/owner=external-dns-gke,..."
```

### TTL Override (optional)

```
external-dns.alpha.kubernetes.io/ttl: "60"
```

---

## DNS Record Lifecycle

```
Service created with annotation
        │
        ▼
Load balancer controller assigns IP
(GCP Cloud LB on GKE, MetalLB on kind)
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

The `txtOwnerId` value is embedded in this record. External-dns will only modify or delete records whose TXT owner matches its own configured `txtOwnerId`.

This matters when both environments (GKE and kind) point at the same Cloud DNS zone simultaneously. Because GKE uses `txtOwnerId: external-dns-gke` and kind uses `txtOwnerId: external-dns-kind`, each instance only manages its own records and will not interfere with the other's. Without distinct owner IDs, either instance would delete records created by the other whenever those Services do not exist in its own cluster.

---

## Verification and Troubleshooting

### Check pods are running

```bash
# external-dns
kubectl get pods -n external-dns

# MetalLB (kind only)
kubectl get pods -n metallb-system
```

### Watch external-dns logs

```bash
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns -f
```

Successful sync:

```json
{"level":"info","msg":"Desired change: CREATE gateway.safabayar.net A [172.18.255.200]"}
{"level":"info","msg":"Desired change: CREATE gateway.safabayar.net TXT [...]"}
{"level":"info","msg":"2 record(s) in zone safabayar.net. were successfully updated"}
```

No annotated Services yet:

```json
{"level":"info","msg":"All records are already up to date"}
```

### Verify records in Cloud DNS

```bash
gcloud dns record-sets list --zone=$DNS_ZONE --project=$PROJECT_ID
```

### Common issues

| Symptom | Environment | Likely cause | Fix |
|---|---|---|---|
| `Permission denied` calling Cloud DNS API | Both | GCP SA lacks `roles/dns.admin` | Re-run step 3 of GCP prerequisites |
| Pod running but logs show no changes, record missing | Both | Service has no annotation | Add `external-dns.alpha.kubernetes.io/hostname` |
| Record not deleted after Service removed | Both | `policy: upsert-only` | Change to `policy: sync` or delete manually |
| `required env var GCP_PROJECT_ID is not set` | Both | Env var not exported | `export GCP_PROJECT_ID=...` |
| Workload Identity annotation missing on K8s SA | GKE | `GCP_PROJECT_ID` not set when `helmfile sync` ran | Re-run `helmfile sync -e gke` with var set |
| Service stuck in `<pending>` | kind | MetalLB not installed or config not applied | Run `helmfile sync -e kind` then `kubectl apply -f metallb-config.yaml` |
| `credentials.json: no such file or directory` | kind | Secret not created before pod started | Create Secret, then `kubectl delete pod -n external-dns -l app.kubernetes.io/name=external-dns` |
| MetalLB assigns IP but it's unreachable | kind | Pool range outside Docker subnet | Check `docker network inspect kind`, update `metallb-config.yaml` |
| TXT ownership conflict, records deleted by wrong instance | Both | GKE and kind share same `txtOwnerId` | Confirm `external-dns-gke.yaml` and `external-dns-kind.yaml` have different `txtOwnerId` values |
