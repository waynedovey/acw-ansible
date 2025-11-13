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

# RHCL (Kuadrant) Geo DNS with Existing Gateway — JSON API Demo (PodSecurity‑safe)

This guide enables **Geo DNS** across two OpenShift clusters (AWS **ap-northeast-2** & **ap-northeast-3**) using **Red Hat Connectivity Link (RHCL / Kuadrant)**, **Gateway API**, and your existing **Gateway** `ingress-gateway/prod-web` (listener `api`).

It deploys a small **JSON API** (`geoapi`) with `/api/v1/info` and `/health` endpoints, designed to be **PodSecurity (restricted)** friendly and to participate in **automatic DNS failover** via RHCL health checks.

**Image:** `quay.io/wdovey/go-httpbin:latest` (public).

---

## What you get

A single wildcard hostname `*.travels.sandbox802.opentlc.com` that geo-routes requests:

```
*.travels…  →  CNAME  klb.travels…
klb.travels… →  Geo CNAMEs:  JP → jp.klb…,  KR → kr.klb…,  default → <default site>
jp.klb…     →  ELB of site-c (ap-northeast-3)
kr.klb…     →  ELB of site-b (ap-northeast-2)
```

RHCL keeps each CNAME **single-valued** (Route 53 requirement) while providing GEO routing and **health-checked failover**.

---

## Prerequisites

- **RHCL operator (Kuadrant)** installed on both clusters.
- **Gateway API v1 CRDs** present on both clusters.
- **GatewayClass** provided by your Mesh/Ingress (e.g., `istio`) installed.
- Existing **Gateway** (already present):
  - Namespace: `ingress-gateway`
  - Name: `prod-web`
  - Listener: `api`
  - Hostname: `*.travels.sandbox802.opentlc.com`
  - TLS: Terminate (secret `api-tls`)
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
    # optional:
    AWS_REGION: <base64 of ap-northeast-2>
  ```

> Tip: restrict the IAM user to the hosted zone for least privilege.

---

## Deploy DNSPolicy via Ansible (per cluster)

Pass GEO and default flags **from the CLI**. Do **not** quote booleans.

### Site‑c (Osaka, ap‑northeast‑3) — make default (example)
```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=JP \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=true \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120
```

### Site‑b (Seoul, ap‑northeast‑2)
```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=KR \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=false \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120
```

**Values consumed by the chart:**
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

## Deploy the JSON API app (both clusters)

App listens on **8080** (non‑privileged) and the Service exposes **80 → targetPort 8080**. No fixed UID; OpenShift assigns a UID from the namespace’s range.

```bash
# Namespace + Deployment + Service
oc apply -f - <<'YAML'
apiVersion: v1
kind: Namespace
metadata:
  name: geoapi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: geoapi
  namespace: geoapi
spec:
  replicas: 2
  selector:
    matchLabels: { app: geoapi }
  template:
    metadata:
      labels: { app: geoapi }
    spec:
      securityContext:
        seccompProfile: { type: RuntimeDefault }
      containers:
      - name: geoapi
        image: quay.io/wdovey/go-httpbin:latest
        args: ["--port", "8080"]
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          capabilities: { drop: ["ALL"] }
        readinessProbe:
          httpGet: { path: /status/200, port: 8080 }
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /status/200, port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: geoapi
  namespace: geoapi
spec:
  selector: { app: geoapi }
  ports:
  - name: http
    port: 80
    targetPort: 8080
YAML
```

---

## HTTPRoute per site (attach to existing Gateway)

Two rules:
- `/api/v1/info` → rewrites to `/headers` (JSON) and sets `X‑Site` header (`JP` or `KR`).
- `/health` → rewrites to `/status/200` (used by RHCL DNS health checks).

> Use wildcard `hostnames` so health checks work across `jp.klb...` / `kr.klb...` names.

### Site‑c (Osaka, **JP**) route
```bash
oc apply -f - <<'YAML'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: geoapi
  namespace: geoapi
spec:
  parentRefs:
  - name: prod-web
    namespace: ingress-gateway
    sectionName: api
  hostnames:
    - "*.travels.sandbox802.opentlc.com"
  rules:
  - matches:
    - path:
        type: Exact
        value: /api/v1/info
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /headers
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        set:
        - name: X-Site
          value: JP
    backendRefs:
    - name: geoapi
      port: 80
  - matches:
    - path:
        type: Exact
        value: /health
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /status/200
    backendRefs:
    - name: geoapi
      port: 80
YAML
```

### Site‑b (Seoul, **KR**) route
```bash
oc apply -f - <<'YAML'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: geoapi
  namespace: geoapi
spec:
  parentRefs:
  - name: prod-web
    namespace: ingress-gateway
    sectionName: api
  hostnames:
    - "*.travels.sandbox802.opentlc.com"
  rules:
  - matches:
    - path:
        type: Exact
        value: /api/v1/info
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /headers
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        set:
        - name: X-Site
          value: KR
    backendRefs:
    - name: geoapi
      port: 80
  - matches:
    - path:
        type: Exact
        value: /health
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /status/200
    backendRefs:
    - name: geoapi
      port: 80
YAML
```

---

## Validate

```bash
# Route accepted & programmed?
oc -n geoapi get httproute geoapi -o jsonpath='{.status.parents[*].conditions[?(@.type=="Accepted")].status}{"\n"}'
oc -n geoapi get httproute geoapi -o jsonpath='{.status.parents[*].conditions[?(@.type=="Programmed")].status}{"\n"}'

# Hit the JSON API (response includes headers with X-Site)
curl -s https://api.travels.sandbox802.opentlc.com/api/v1/info | jq '.headers["X-Site"]'
curl -sI https://api.travels.sandbox802.opentlc.com/api/v1/info | grep -i ^x-site:

# Health endpoint (should be 200)
curl -si https://api.travels.sandbox802.opentlc.com/health | head -n1
```

**Route 53 view:**
```bash
aws route53 list-resource-record-sets --hosted-zone-id <ZONE_ID> \
  --query "ResourceRecordSets[?contains(Name, 'travels.sandbox802.opentlc.com.')]" \
  --output table
```

---

## Force switchover / failover

### Flip the **default** site (affects non‑geo clients)
```bash
# KR (site-b)
oc -n ingress-gateway patch dnspolicy prod-web-dnspolicy --type=merge -p \
  '{"spec":{"loadBalancing":{"defaultGeo":true}}}'

# JP (site-c)
oc -n ingress-gateway patch dnspolicy prod-web-dnspolicy --type=merge -p \
  '{"spec":{"loadBalancing":{"defaultGeo":false}}}'
```

### Simulate failure on one site (RHCL removes that geo from DNS)
On the site you want to drain, change `/health` rewrite to return 500:
```bash
oc -n geoapi patch httproute geoapi --type=json -p='[
  {"op":"replace","path":"/spec/rules/1/filters/0/urlRewrite/path/replaceFullPath","value":"/status/500"}
]'
```
Watch status & records (remember `klb.*` TTL is typically **300s**):
```bash
oc -n ingress-gateway describe dnspolicy prod-web-dnspolicy | egrep -A2 'Enforced|SubResourcesHealthy'
aws route53 list-resource-record-sets --hosted-zone-id <ZONE_ID> \
  --query "ResourceRecordSets[?Name=='klb.travels.sandbox802.opentlc.com.']" --output table
```

Restore health:
```bash
oc -n geoapi patch httproute geoapi --type=json -p='[
  {"op":"replace","path":"/spec/rules/1/filters/0/urlRewrite/path/replaceFullPath","value":"/status/200"}
]'
```

---

## Troubleshooting

- **SCC/UID error:** do **not** set a fixed `runAsUser`. Let OpenShift assign a UID; keep `runAsNonRoot: true` and `capabilities: { drop: ["ALL"] }`.
- **Bind to port 80 fails:** use **8080** in the container; keep **Service port 80 → targetPort 8080**.
- **Route not accepted/programmed:** check `GatewayClass`, listener `api`, and Gateway API v1 CRDs.
- **DNS not flipping:** verify DNSPolicy status (`Accepted/Enforced/Healthy`), check Route53 records, and wait for the `klb.*` TTL.
- **Argo drift:** persist `defaultGeo` and per‑site `geoCode` in IaC so Git doesn’t revert your change.

---

## Clean up (demo only)
```bash
oc -n geoapi delete httproute geoapi || true
oc -n geoapi delete svc geoapi || true
oc -n geoapi delete deploy geoapi || true
oc delete ns geoapi || true
```
