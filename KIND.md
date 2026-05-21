# External DNS — kind On-Premises Installation

This guide covers deploying external-dns on a [kind](https://kind.sigs.k8s.io/) (Kubernetes IN Docker) cluster with GCP Cloud DNS as the DNS provider. It is a self-contained reference for the `kind` helmfile environment; for the GKE setup see [README.md](README.md).

---

## Table of Contents

1. [Overview](#overview)
2. [How It Works](#how-it-works)
3. [Architecture](#architecture)
4. [Component Roles](#component-roles)
5. [Prerequisites](#prerequisites)
6. [GCP Setup](#gcp-setup)
7. [Create the kind Cluster](#create-the-kind-cluster)
8. [Install MetalLB](#install-metallb)
   - [Why MetalLB Is Needed](#why-metallb-is-needed)
   - [How MetalLB Works in kind](#how-metallb-works-in-kind)
   - [Configure the IP Pool](#configure-the-ip-pool)
9. [Install External DNS](#install-external-dns)
   - [Create the Credentials Secret](#create-the-credentials-secret)
   - [Deploy with Helmfile](#deploy-with-helmfile)
10. [Verify the Full Stack](#verify-the-full-stack)
11. [Annotate Services for DNS](#annotate-services-for-dns)
    - [Example: Gateway Service](#example-gateway-service)
12. [DNS Record Lifecycle](#dns-record-lifecycle)
13. [Teardown](#teardown)
14. [Troubleshooting](#troubleshooting)

---

## Overview

The kind environment installs two components that GKE handles transparently:

- **MetalLB** — assigns real IP addresses to `LoadBalancer` Services inside the Docker network, because kind has no cloud load balancer controller.
- **GCP Service Account JSON key** — authenticates external-dns to Cloud DNS, because kind has no GKE metadata server and therefore cannot use Workload Identity.

Everything else — the external-dns reconciliation loop, the Cloud DNS records, the annotation-based opt-in — works identically to GKE.

---

## How It Works

```
┌─ Every 1 minute ─────────────────────────────────────────────────────────┐
│                                                                           │
│  1. external-dns queries the K8s API for all LoadBalancer Services       │
│     across all namespaces.                                                │
│                                                                           │
│  2. For each Service that has the annotation:                             │
│       external-dns.alpha.kubernetes.io/hostname: foo.safabayar.net        │
│     external-dns reads status.loadBalancer.ingress[0].ip                  │
│     (the IP that MetalLB assigned from the kind-pool).                    │
│                                                                           │
│  3. external-dns calls the GCP Cloud DNS API (authenticated via the       │
│     credentials.json key mounted from the K8s Secret) and upserts:        │
│       foo.safabayar.net  A    → 172.18.255.200                            │
│       foo.safabayar.net  TXT  → "heritage=external-dns,owner=..."         │
│                                                                           │
│  4. If a previously managed Service no longer exists or its annotation    │
│     was removed, external-dns deletes the corresponding A + TXT records.  │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture

```
Host machine
│
│  docker network: kind (172.18.0.0/16)
│  ┌─────────────────────────────────────────────────────────────────────┐
│  │  kind cluster                                                        │
│  │                                                                      │
│  │  ┌─────────────────────┐    ┌──────────────────────────────────────┐ │
│  │  │  namespace: default │    │  namespace: metallb-system           │ │
│  │  │                     │    │                                      │ │
│  │  │  Service (LB)       │    │  controller (watches for LB svcs)   │ │
│  │  │  ┌───────────────┐  │    │  speaker (ARP on Docker network)    │ │
│  │  │  │ type:         │  │    │                                      │ │
│  │  │  │  LoadBalancer │  │    │  IPAddressPool: 172.18.255.200-250   │ │
│  │  │  │               │◄─┼────┼──────── assigns IP from pool ───────┘ │
│  │  │  │ annotation:   │  │    │                                        │
│  │  │  │  hostname:    │  │    │  ┌──────────────────────────────────┐  │
│  │  │  │  foo.example  │  │    │  │  namespace: external-dns         │  │
│  │  │  │               │  │    │  │                                  │  │
│  │  │  │ status:       │◄─┼────┼──│  Pod: external-dns               │  │
│  │  │  │  LB IP:       │  │    │  │  - reads Service annotations     │  │
│  │  │  │  172.18.255.x │  │    │  │  - reads MetalLB-assigned IPs    │  │
│  │  │  └───────────────┘  │    │  │  - mounts credentials.json       │  │
│  │  └─────────────────────┘    │  │    from K8s Secret               │  │
│  │                             │  └────────────┬─────────────────────┘  │
│  └─────────────────────────────┴───────────────┼────────────────────────┘
│                                                │
│  ARP: MetalLB speaker advertises               │ HTTPS (outbound)
│       172.18.255.x on the Docker network       │
│  → host can reach those IPs directly           │
│                                                ▼
│                               ┌─────────────────────────────┐
│                               │  GCP Cloud DNS              │
│                               │                             │
│                               │  foo.safabayar.net  A       │
│                               │    → 172.18.255.200         │
│                               │                             │
│                               │  foo.safabayar.net  TXT     │
│                               │    → "heritage=..."         │
│                               └─────────────────────────────┘
```

---

## Component Roles

### kind

Creates a multi-node Kubernetes cluster entirely inside Docker containers. Each node is a container connected to a Docker bridge network. kind provides a full Kubernetes API but no cloud integrations — there is no cloud load balancer controller, no cloud storage provisioner, and no identity federation.

### MetalLB

A software load balancer for bare-metal (and kind) Kubernetes clusters. It has two components:

- **controller** — watches for `LoadBalancer` Services and allocates IPs from the configured pool.
- **speaker** — advertises those IPs on the network. In L2 mode (used here) it responds to ARP requests on the Docker network, making the assigned IPs reachable from the host machine and between kind nodes.

Without MetalLB, every `LoadBalancer` Service in kind stays in `<pending>` state because nothing allocates or advertises the external IP. External-dns waits for `status.loadBalancer.ingress` to be populated before it creates a DNS record — so if IPs never appear, no records are ever created.

### External DNS

Watches Kubernetes Services for the `external-dns.alpha.kubernetes.io/hostname` annotation. When a Service has the annotation and a populated LoadBalancer IP, external-dns calls the GCP Cloud DNS API to create or update the corresponding `A` record. It also creates a `TXT` record to record ownership, allowing it to clean up records when the Service is deleted.

### GCP Cloud DNS

The authoritative DNS zone. External-dns writes to it directly via the Cloud DNS API. The DNS records created on a kind cluster point to private RFC1918 addresses (`172.18.255.x`) — they resolve correctly from the host machine and inside the Docker network but are not routable from the public internet. This makes kind suitable for testing the full DNS automation workflow locally without needing public IPs.

---

## Prerequisites

| Tool | Minimum version | Install |
|---|---|---|
| Docker | 20.x | https://docs.docker.com/get-docker/ |
| kind | 0.20+ | `brew install kind` |
| kubectl | 1.28+ | `brew install kubectl` |
| helm | 3.x | `brew install helm` |
| helmfile | 0.162+ | `brew install helmfile` |
| gcloud | any | https://cloud.google.com/sdk/docs/install |

Install the helm-diff plugin, which helmfile requires to show changes before applying:

```bash
helm plugin install https://github.com/databus23/helm-diff
```

---

## GCP Setup

External-dns on kind still writes to GCP Cloud DNS. A GCP service account with DNS admin permissions is required. If you already created this for GKE, skip to [Create the kind Cluster](#create-the-kind-cluster).

```bash
export PROJECT_ID="your-gcp-project-id"
export DNS_ZONE="safabayar-net"    # managed zone name, not the domain

# 1. Enable the Cloud DNS API
gcloud services enable dns.googleapis.com --project=$PROJECT_ID

# 2. Create the service account
gcloud iam service-accounts create external-dns \
  --display-name="External DNS" \
  --project=$PROJECT_ID

# 3. Grant DNS admin
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:external-dns@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/dns.admin"

# 4. Download a JSON key — this file is the cluster credential
gcloud iam service-accounts keys create external-dns-sa-key.json \
  --iam-account=external-dns@$PROJECT_ID.iam.gserviceaccount.com \
  --project=$PROJECT_ID

# 5. Keep the key out of git
echo "external-dns-sa-key.json" >> .gitignore

# 6. Verify your Cloud DNS zone exists (create if not)
gcloud dns managed-zones describe $DNS_ZONE --project=$PROJECT_ID \
  || gcloud dns managed-zones create $DNS_ZONE \
       --description="Public zone for safabayar.net" \
       --dns-name="safabayar.net." \
       --visibility=public \
       --project=$PROJECT_ID
```

The JSON key file grants whoever holds it full DNS admin access. Treat it like a password: do not commit it, do not share it, and rotate it with `gcloud iam service-accounts keys create` + `gcloud iam service-accounts keys delete` if it is ever exposed.

---

## Create the kind Cluster

`kind-cluster.yaml` at the root of this repository defines the cluster topology:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
networking:
  disableDefaultCNI: false
```

Create the cluster:

```bash
kind create cluster \
  --config kind-cluster.yaml \
  --name external-dns-dev

# Verify nodes are Ready
kubectl get nodes
# NAME                            STATUS   ROLES           AGE   VERSION
# external-dns-dev-control-plane  Ready    control-plane   60s   v1.31.x
# external-dns-dev-worker         Ready    <none>          40s   v1.31.x
```

kubectl's context is set automatically to `kind-external-dns-dev`. All subsequent `kubectl` commands target this cluster.

---

## Install MetalLB

### Why MetalLB Is Needed

When you create a `Service` of type `LoadBalancer` in kind, Kubernetes writes the Service object and waits for a controller to fill in `status.loadBalancer.ingress`. In GKE that controller is the GCP cloud controller manager, which provisions a real load balancer and returns its IP. In kind there is no such controller, so `status.loadBalancer.ingress` stays empty and the Service shows `<pending>` under `EXTERNAL-IP`.

External-dns polls `status.loadBalancer.ingress` on every sync cycle. If the field is empty it logs `skipping Service — no IP assigned yet` and moves on. No IP means no DNS record is ever created, regardless of whether the annotation is present.

MetalLB solves this by acting as that missing controller inside the cluster.

### How MetalLB Works in kind

MetalLB runs two components:

**controller** — watches the Kubernetes API for `LoadBalancer` Services. When it sees one without an IP, it allocates an address from the configured `IPAddressPool` and writes it into `status.loadBalancer.ingress`.

**speaker** — runs as a `DaemonSet` on every node. In L2 mode it listens on all node network interfaces and responds to ARP requests for the IPs the controller has allocated. Because kind nodes are Docker containers connected to the Docker bridge network (`172.18.0.0/16` by default), the speaker's ARP responses are visible on that bridge — which means the host machine and all kind nodes can reach those IPs directly.

```
Host asks: "Who has 172.18.255.200?"
  │
  ▼ Docker bridge network
MetalLB speaker (running on kind-worker node container) responds:
  "I have 172.18.255.200 — my MAC is xx:xx:xx:xx:xx:xx"
  │
  ▼
Traffic to 172.18.255.200 is forwarded to the kind-worker container
  │
  ▼
kube-proxy routes it to the Service's backend pod
```

### Configure the IP Pool

The `IPAddressPool` range must fall within the Docker bridge subnet that kind uses. MetalLB allocates addresses from this range; kind nodes themselves use addresses at the low end of the subnet, so allocating from the high end avoids collisions.

**Step 1 — Find your Docker kind network subnet:**

```bash
docker network inspect kind \
  --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
# Example: 172.18.0.0/16
```

**Step 2 — Check `metallb-config.yaml` matches:**

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: kind-pool
  namespace: metallb-system
spec:
  addresses:
    - 172.18.255.200-172.18.255.250   # adjust if your subnet differs
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: kind-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - kind-pool
```

If your Docker network uses `172.19.0.0/16` instead, change the pool to `172.19.255.200-172.19.255.250`.

`L2Advertisement` tells MetalLB to use ARP (L2 mode) to announce the allocated IPs. This is the correct mode for kind because there is no BGP router available.

---

## Install External DNS

### Create the Credentials Secret

The Secret must exist in the `external-dns` namespace before the external-dns pod starts. If the pod starts without the Secret, it exits immediately. Create the namespace and Secret manually:

```bash
kubectl create namespace external-dns

kubectl create secret generic external-dns-gcp-credentials \
  --namespace external-dns \
  --from-file=credentials.json=./external-dns-sa-key.json
```

Verify it was created correctly:

```bash
kubectl get secret -n external-dns external-dns-gcp-credentials
# NAME                           TYPE     DATA   AGE
# external-dns-gcp-credentials   Opaque   1      5s

# Confirm the key inside (should show the key file name, not the content)
kubectl get secret -n external-dns external-dns-gcp-credentials \
  -o jsonpath='{.data}' | python3 -c "import sys,json; d=json.load(sys.stdin); print(list(d.keys()))"
# ['credentials.json']
```

The chart mounts this Secret at a well-known path and points the GCP client library at it via the `GOOGLE_APPLICATION_CREDENTIALS` environment variable. The relevant chart values are set in `values/external-dns-kind.yaml`:

```yaml
google:
  serviceAccountSecret: external-dns-gcp-credentials
  serviceAccountSecretKey: credentials.json
```

### Deploy with Helmfile

```bash
cd /path/to/external-dns-k8s

export GCP_PROJECT_ID="your-gcp-project-id"

# Preview — shows exactly what Kubernetes resources will be created
helmfile diff -e kind

# Deploy MetalLB + external-dns
helmfile sync -e kind
```

`helmfile sync -e kind` does the following in order:

1. Adds the `external-dns` and `metallb` Helm repositories if not already present.
2. Installs the `metallb` chart into `metallb-system` (because `metallb.enabled: true` in `environments/kind.yaml`).
3. Installs the `external-dns` chart into `external-dns`, merging `values/external-dns.yaml` (common) and `values/external-dns-kind.yaml` (kind overlay), and injecting `google.project` from `GCP_PROJECT_ID`.

**After `helmfile sync` completes, apply the MetalLB pool config:**

The `IPAddressPool` and `L2Advertisement` are custom resources whose CRDs are installed by the MetalLB chart. They cannot be applied until those CRDs exist. Wait for MetalLB to be ready first:

```bash
kubectl rollout status deployment -n metallb-system controller --timeout=90s
kubectl rollout status daemonset  -n metallb-system speaker   --timeout=90s

# Apply the IP pool and advertisement config
kubectl apply -f metallb-config.yaml
```

Expected output:

```
ipaddresspool.metallb.io/kind-pool created
l2advertisement.metallb.io/kind-l2 created
```

---

## Verify the Full Stack

Run these checks in order to confirm every layer is working before annotating real Services.

### 1. MetalLB pods are Running

```bash
kubectl get pods -n metallb-system
# NAME                          READY   STATUS    RESTARTS   AGE
# controller-xxxxxxxxx-xxxxx    1/1     Running   0          2m
# speaker-xxxxx                 1/1     Running   0          2m
# speaker-yyyyy                 1/1     Running   0          2m
```

### 2. MetalLB assigns IPs to LoadBalancer Services

Create a throwaway Service to confirm MetalLB is allocating IPs:

```bash
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80 --type=LoadBalancer

# Wait a few seconds, then:
kubectl get svc nginx
# NAME    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
# nginx   LoadBalancer   10.96.x.x      172.18.255.200   80:3xxxx/TCP   10s

# Confirm reachability from host
curl -s -o /dev/null -w "%{http_code}" http://172.18.255.200
# 200

# Clean up
kubectl delete pod nginx
kubectl delete svc nginx
```

If `EXTERNAL-IP` shows `<pending>`, MetalLB is not assigning IPs. See [Troubleshooting](#troubleshooting).

### 3. External-dns pod is Running

```bash
kubectl get pods -n external-dns
# NAME                            READY   STATUS    RESTARTS   AGE
# external-dns-xxxxxxxxx-xxxxx    1/1     Running   0          2m
```

### 4. External-dns can authenticate to Cloud DNS

```bash
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns --tail=20
```

Look for the startup line confirming the provider loaded:

```json
{"level":"info","msg":"Connected to cluster"}
{"level":"info","msg":"All records are already up to date"}
```

If you see `Permission denied` or `invalid_grant`, the credentials are wrong or the service account lacks `roles/dns.admin`. See [Troubleshooting](#troubleshooting).

### 5. End-to-end: annotate a Service and watch the record appear

```bash
# Create a test LoadBalancer Service with the external-dns annotation
kubectl create deployment test-app --image=nginx
kubectl expose deployment test-app --port=80 --type=LoadBalancer \
  --overrides='{"metadata":{"annotations":{"external-dns.alpha.kubernetes.io/hostname":"test.safabayar.net"}}}'

# Watch external-dns logs for the sync
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns -f
```

Within one reconciliation interval (default 1 minute) you should see:

```json
{"level":"info","msg":"Desired change: CREATE test.safabayar.net A [172.18.255.201]"}
{"level":"info","msg":"Desired change: CREATE test.safabayar.net TXT [...]"}
{"level":"info","msg":"2 record(s) in zone safabayar.net. were successfully updated"}
```

Verify the record in Cloud DNS:

```bash
gcloud dns record-sets list --zone=$DNS_ZONE --project=$PROJECT_ID \
  --filter="name=test.safabayar.net."
```

Clean up:

```bash
kubectl delete deployment test-app
kubectl delete svc test-app
# external-dns will delete the DNS record on the next sync cycle
```

---

## Annotate Services for DNS

External-dns ignores Services without the hostname annotation. Add it to opt in:

```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: foo.safabayar.net
```

Multiple hostnames (comma-separated):

```yaml
external-dns.alpha.kubernetes.io/hostname: foo.safabayar.net,bar.safabayar.net
```

Custom TTL (default is 300 seconds):

```yaml
external-dns.alpha.kubernetes.io/ttl: "60"
```

### Example: Gateway Service

The gateway project (`/root/poc/gateway`) uses a Helm chart. Add the annotation via a values override:

```yaml
# gateway-kind-override.yaml
service:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: gateway.safabayar.net
```

Apply:

```bash
helm upgrade gateway /root/poc/gateway/helm/gateway \
  -f /root/poc/gateway/helm/gateway/values.yaml \
  -f gateway-kind-override.yaml
```

Or inline:

```bash
helm upgrade gateway /root/poc/gateway/helm/gateway \
  --set 'service.annotations.external-dns\.alpha\.kubernetes\.io/hostname=gateway.safabayar.net'
```

After MetalLB assigns the IP and external-dns runs its next sync, the record appears:

```
gateway.safabayar.net.  300  IN  A    172.18.255.200
gateway.safabayar.net.  300  IN  TXT  "heritage=external-dns,external-dns/owner=external-dns-kind,..."
```

Note that the `TXT` record carries `owner=external-dns-kind` — this is the `txtOwnerId` set in `values/external-dns-kind.yaml`. It is intentionally different from the GKE value (`external-dns-gke`) so that if both environments target the same Cloud DNS zone, neither will delete the other's records.

---

## DNS Record Lifecycle

```
kubectl apply / helm upgrade
  │
  │  Service of type LoadBalancer created or updated
  ▼
MetalLB controller
  │
  │  Allocates IP from kind-pool (172.18.255.200-250)
  │  Writes IP to status.loadBalancer.ingress[0].ip
  ▼
MetalLB speaker
  │
  │  Announces IP via ARP on the Docker bridge network
  │  → host and other nodes can reach the IP
  ▼
external-dns reconciliation loop (fires every 1 minute)
  │
  │  Reads Service annotation:
  │    external-dns.alpha.kubernetes.io/hostname: foo.safabayar.net
  │  Reads Service IP: 172.18.255.200
  ▼
GCP Cloud DNS API
  │
  ├── Upsert: foo.safabayar.net  A    → 172.18.255.200  (TTL 300)
  └── Upsert: foo.safabayar.net  TXT  → "heritage=external-dns,owner=external-dns-kind,..."


kubectl delete / helm uninstall
  │
  │  Service deleted
  ▼
external-dns reconciliation loop (next cycle)
  │
  │  Service no longer exists
  │  TXT record owner matches txtOwnerId (external-dns-kind)
  ▼
GCP Cloud DNS API
  │
  ├── Delete: foo.safabayar.net  A
  └── Delete: foo.safabayar.net  TXT
```

---

## Teardown

### Remove external-dns and MetalLB

```bash
helmfile destroy -e kind
```

This uninstalls both Helm releases. DNS records that were already created are **not** automatically deleted by `helmfile destroy` — they would have been deleted on external-dns's next sync cycle, but the pod is gone before that cycle runs. Delete orphaned records manually:

```bash
gcloud dns record-sets list --zone=$DNS_ZONE --project=$PROJECT_ID
gcloud dns record-sets delete foo.safabayar.net \
  --type=A --zone=$DNS_ZONE --project=$PROJECT_ID
gcloud dns record-sets delete foo.safabayar.net \
  --type=TXT --zone=$DNS_ZONE --project=$PROJECT_ID
```

### Delete the kind cluster

```bash
kind delete cluster --name external-dns-dev
```

### Revoke the JSON key (recommended)

```bash
# List key IDs for the service account
gcloud iam service-accounts keys list \
  --iam-account=external-dns@$PROJECT_ID.iam.gserviceaccount.com \
  --project=$PROJECT_ID

# Delete the key by ID
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=external-dns@$PROJECT_ID.iam.gserviceaccount.com \
  --project=$PROJECT_ID
```

---

## Troubleshooting

### Service stuck in `<pending>` — MetalLB not assigning IPs

```bash
# Check MetalLB controller logs
kubectl logs -n metallb-system -l app=metallb,component=controller

# Check that the IPAddressPool was applied
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

If `ipaddresspool` returns `No resources found`, the MetalLB CRDs were not ready when `metallb-config.yaml` was applied. Re-apply:

```bash
kubectl rollout status deployment -n metallb-system controller --timeout=90s
kubectl apply -f metallb-config.yaml
```

If the pool exists but IPs are still not assigned, check that the pool range is within the Docker kind network subnet:

```bash
docker network inspect kind --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

### External-dns pod crashes — `credentials.json: no such file or directory`

The Secret was not created before the pod started. Create it and restart the pod:

```bash
kubectl create secret generic external-dns-gcp-credentials \
  --namespace external-dns \
  --from-file=credentials.json=./external-dns-sa-key.json

kubectl delete pod -n external-dns \
  -l app.kubernetes.io/name=external-dns
```

### External-dns logs `invalid_grant` or `Permission denied`

The JSON key is invalid, expired, or the service account was deleted. Generate a new key:

```bash
gcloud iam service-accounts keys create external-dns-sa-key-new.json \
  --iam-account=external-dns@$PROJECT_ID.iam.gserviceaccount.com \
  --project=$PROJECT_ID

kubectl create secret generic external-dns-gcp-credentials \
  --namespace external-dns \
  --from-file=credentials.json=./external-dns-sa-key-new.json \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl delete pod -n external-dns \
  -l app.kubernetes.io/name=external-dns
```

### DNS record created but `A` record points to wrong IP

External-dns self-corrects. If MetalLB reallocates a different IP (e.g., after a cluster restart), external-dns detects the mismatch on the next sync and updates the record. No manual action is needed — wait up to 1 minute.

### `required env var GCP_PROJECT_ID is not set`

```bash
export GCP_PROJECT_ID="your-gcp-project-id"
helmfile sync -e kind
```

### TXT ownership conflict — records unexpectedly deleted

If the GKE environment and the kind environment share the same Cloud DNS zone with the same `txtOwnerId`, one instance will delete the other's records. Confirm:

```bash
# GKE should have:
grep txtOwnerId values/external-dns-gke.yaml
# txtOwnerId: external-dns-gke

# kind should have:
grep txtOwnerId values/external-dns-kind.yaml
# txtOwnerId: external-dns-kind
```

If they are the same, change one of them and redeploy that environment.

### `helmfile sync` fails with `Error: repository not found`

```bash
# Re-add repositories
helmfile repos -e kind

# Then retry
helmfile sync -e kind
```
