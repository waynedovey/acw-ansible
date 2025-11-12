To run the playbook:

## TLDR:

```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml -e ACTION=create -e ocp4_workload_app_connectivity_workshop_acme_email="wdovey@gmail.com"
```

Example runs for Region options

Osaka / ap-northeast-3 (JP, default = true) site-c

```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=JP \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=true \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120
```

Seoul / ap-northeast-2 (KR, default = false) site-b 

```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=KR \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=false \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120
```

# RHCL (Kuadrant) Geo DNS with Existing Gateway

This guide enables **Geo DNS** across two OpenShift clusters (AWS **ap-northeast-2** & **ap-northeast-3**) using **Red Hat Connectivity Link (RHCL / Kuadrant)**, **Gateway API**, and your existing **Gateway** `ingress-gateway/prod-web` (listener `api`).

It assumes this repo drives your Argo CD apps and Helm values for platform charts (ingress-gateway, RHCL operator, Service Mesh, etc.).

---

## What you get

A single wildcard hostname `*.travels.sandbox802.opentlc.com` that geo-routes:

```
*.travels…  →  CNAME  klb.travels…
klb.travels… →  Geo CNAMEs:  JP → jp.klb…,  KR → kr.klb…,  Default → jp.klb…
jp.klb…     →  ELB of site-c (ap-northeast-3)
kr.klb…     →  ELB of site-b (ap-northeast-2)
```

RHCL keeps each CNAME **single-valued** (Route 53 requirement) while still providing GEO routing and health-checked failover.

---

## Prerequisites

- **RHCL operator (Kuadrant)** installed on both clusters.
- **Gateway API v1 CRDs** present on both clusters.
- **GatewayClass** provided by your Mesh/Ingress (e.g., `istio`) installed.
- Existing **Gateway**:
  - Namespace: `ingress-gateway`
  - Name: `prod-web`
  - Listener: `api`
  - Hostname: `*.travels.sandbox802.opentlc.com`
- **Route 53 credentials secret** in each cluster/namespace:
  ```yaml
  apiVersion: v1
  kind: Secret
  type: kuadrant.io/aws
  metadata:
    name: prod-web-aws-credentials
    namespace: ingress-gateway
  data:
    AWS_ACCESS_KEY_ID: <base64>
    AWS_SECRET_ACCESS_KEY: <base64>
    # optional but recommended
    AWS_REGION: <base64 of ap-northeast-2>
  ```

> Tip: restrict the IAM user to the specific hosted zone for least privilege.

---

## Deploy via Ansible (per cluster)

You pass GEO and default flags **from the CLI**. Do **not** quote booleans.

### Site‑c (Osaka, ap-northeast‑3) — **Default**
```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=JP \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=true \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120
```

### Site‑b (Seoul, ap-northeast‑2)
```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=KR \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=false \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120
```

**Helm values shape consumed by the chart:**
```yaml
dns:
  routingStrategy: loadbalanced
  loadBalancing:
    geoCode: "JP"|"KR"
    defaultGeo: true|false
    weight: 120
  healthCheck:
    path: /health
    interval: 1m
    failureThreshold: 3
```

---

## Validate on each cluster

### 1) Gateway has an external address
```bash
oc -n ingress-gateway get gateway prod-web -o=jsonpath='{.status.addresses[*].value}{"\n"}'
# Expect an AWS ELB hostname
```

### 2) DNSPolicy enforced & healthy
```bash
oc -n ingress-gateway get dnspolicy prod-web-dnspolicy -o yaml | yq '.status'
# Expect:
# - Accepted=True
# - Enforced=True
# - SubResourcesHealthy=True
```

---

## Validate in Route 53

List all relevant records:
```bash
aws route53 list-resource-record-sets --hosted-zone-id <ZONE_ID> \
  --query "ResourceRecordSets[?contains(Name, 'travels.sandbox802.opentlc.com.')]" \
  --output table
```
You should see:
- `*.travels.sandbox802.opentlc.com.` → `CNAME klb.travels.sandbox802.opentlc.com.`
- `klb.travels.sandbox802.opentlc.com.` with Geo entries for `JP`, `KR`, and `default`
- `jp.klb…` → ELB for ap‑northeast‑3, `kr.klb…` → ELB for ap‑northeast‑2
- TXT ownership markers created by Kuadrant/external‑dns

---

## End‑to‑end DNS checks

From anywhere:
```bash
dig whoami.travels.sandbox802.opentlc.com +short
# Expect: whoami → klb → jp.klb (or kr.klb) → correct ELB by geo
```

From inside each cluster (one‑shot pod; may need PodSecurity‑friendly spec):
```bash
# JP cluster:
oc -n ingress-gateway run dnscheck --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup whoami.travels.sandbox802.opentlc.com 1.1.1.1

# KR cluster:
oc -n ingress-gateway run dnscheck --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup whoami.travels.sandbox802.opentlc.com 1.1.1.1
```

> If PodSecurity blocks `oc run`, use the demo curl pod below and `oc exec` into it.

---

## Optional: Demo app with **X‑Site** header

Deploy the same app to both clusters; only the **HTTPRoute** header differs (JP vs KR).

### Common app (both clusters)
```bash
oc apply -f - <<'YAML'
apiVersion: v1
kind: Namespace
metadata:
  name: whoami
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: whoami
spec:
  replicas: 2
  selector: { matchLabels: { app: whoami } }
  template:
    metadata: { labels: { app: whoami } }
    spec:
      containers:
      - name: whoami
        image: ghcr.io/traefik/whoami:v1.10.2
        ports: [{ containerPort: 80 }]
        readinessProbe: { httpGet: { path: /, port: 80 }, initialDelaySeconds: 2, periodSeconds: 5 }
        livenessProbe:  { httpGet: { path: /, port: 80 }, initialDelaySeconds: 5, periodSeconds: 10 }
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: whoami
spec:
  selector: { app: whoami }
  ports: [{ name: http, port: 80, targetPort: 80 }]
YAML
```

### HTTPRoute on site‑c (JP, default)
```bash
oc apply -f - <<'YAML'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: whoami
  namespace: whoami
spec:
  parentRefs:
    - name: prod-web
      namespace: ingress-gateway
      sectionName: api
  hostnames:
    - whoami.travels.sandbox802.opentlc.com
  rules:
    - backendRefs: [{ name: whoami, port: 80 }]
      filters:
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            set: [{ name: X-Site, value: JP }]
YAML
```

### HTTPRoute on site‑b (KR)
```bash
oc apply -f - <<'YAML'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: whoami
  namespace: whoami
spec:
  parentRefs:
    - name: prod-web
      namespace: ingress-gateway
      sectionName: api
  hostnames:
    - whoami.travels.sandbox802.opentlc.com
  rules:
    - backendRefs: [{ name: whoami, port: 80 }]
      filters:
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            set: [{ name: X-Site, value: KR }]
YAML
```

### Validate
```bash
# Route attached?
oc -n whoami get httproute whoami -o jsonpath='{.status.parents[*].conditions[?(@.type=="Accepted")].status}{"\n"}'

# From your laptop:
curl -sI https://whoami.travels.sandbox802.opentlc.com | grep -i ^x-site:

# PodSecurity‑friendly curl pod:
oc -n whoami apply -f - <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: curl
  namespace: whoami
spec:
  securityContext:
    seccompProfile: { type: RuntimeDefault }
  containers:
  - name: curl
    image: registry.access.redhat.com/ubi9/ubi-minimal:9
    command: ["sleep","3600"]
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      capabilities: { drop: ["ALL"] }
YAML

oc -n whoami exec -it pod/curl -- bash -lc 'microdnf -y install curl >/dev/null 2>&1; curl -sI https://whoami.travels.sandbox802.opentlc.com | grep -i ^x-site:'
```

Expected:
- `X-Site: JP` when routed to **site‑c (ap‑northeast‑3)**
- `X-Site: KR` when routed to **site‑b (ap‑northeast‑2)**

---

## Health‑check & failover

You enabled:
```yaml
healthCheck:
  path: /health
  interval: 1m
  failureThreshold: 3
  protocol: HTTPS
  port: 443
```

**Test failover:**
1. Break `/health` on one site (scale app to 0 or return 500).
2. Watch policy & records:
   ```bash
   oc -n ingress-gateway describe dnspolicy prod-web-dnspolicy
   aws route53 list-resource-record-sets --hosted-zone-id <ZONE_ID> \
     --query "ResourceRecordSets[?contains(Name, 'klb.travels.sandbox802.opentlc.com.')]" --output table
   ```
3. After a few intervals, Route 53 stops returning that site’s CNAME for its geo.

---

## Troubleshooting

- **Gateway missing (one site):**
  - Ensure **Gateway API v1 CRDs** exist:
    ```bash
    oc get crd gateways.gateway.networking.k8s.io -o jsonpath='{.spec.versions[*].name}{"\n"}'
    ```
  - Confirm **GatewayClass** (e.g., `istio`) exists:
    ```bash
    oc get gatewayclass
    ```
  - Make sure the **CRD app syncs before** the ingress‑gateway app (use Argo **sync waves**).

- **DNSPolicy Enforced but no `dnsrecord` CRs:**
  - That’s fine. Kuadrant may not emit per‑record CRs. Use **DNSPolicy status** and **Route 53** records.

- **CNAME with multiple values error:**
  - Ensure **all clusters** use **load‑balanced** `DNSPolicy` and only **one** has `defaultGeo: true`.
  - Remove any manually‑created conflicting wildcard in the zone.

---

## Clean up (demo only)
```bash
oc -n whoami delete httproute whoami || true
oc -n whoami delete svc whoami || true
oc -n whoami delete deploy whoami || true
oc delete ns whoami || true
```
