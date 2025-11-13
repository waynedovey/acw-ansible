# RHCL (Kuadrant) Geo DNS with Existing Gateway — **Per‑Cluster Runbook** (JP & KR)

This runbook deploys a **geo-routed, health-checked** API across **two OpenShift clusters** using **Red Hat Connectivity Link (RHCL/Kuadrant)**, the **Gateway API**, and your **existing Gateway**.  
It’s formatted as **one clear section per cluster** so it’s unambiguous.

- **Clusters**
  - **JP** = site‑c (AWS `ap-northeast-3`) → **Default** geo
  - **KR** = site‑b (AWS `ap-northeast-2`)
- **Gateway**: `ingress-gateway/prod-web` (listener `api`, TLS secret `api-tls` covering `*.travels.sandbox802.opentlc.com`)
- **Public host**: `api.travels.sandbox802.opentlc.com` (exact host to avoid wildcard collisions)
- **App**: `geoapi` (public image `quay.io/wdovey/go-httpbin:latest`, listens on **8080**, PodSecurity “restricted” compliant)
- **Endpoints**
  - `/api/v1/info` → rewrites to `/headers`, sets `X-Site` = **JP** or **KR**
  - `/health` → rewrites to `/status/200` (used by DNS health checks)
- **DNS**: Route53 hosted zone `travels.sandbox802.opentlc.com` managed by RHCL via **DNSPolicy**


---

## Prerequisites (both clusters)

1) **RHCL (Kuadrant) installed** and healthy.  
2) **Gateway API v1 CRDs** and `GatewayClass` (e.g., `istio`).  
3) Existing **Gateway** `prod-web` in `ingress-gateway` with listener `api` and TLS secret `api-tls` for `*.travels.sandbox802.opentlc.com`.  
4) **AWS Route53** hosted zone and **credentials** (per cluster) that can manage it:

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
#  AWS_REGION: <base64 of ap-northeast-2>
```

---

## Step 0 — Deploy the app (do this on **each** cluster)

```bash
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
        command: ["/bin/go-httpbin"]
        args: ["-port=8080"]
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

# Cluster JP (site‑c, ap‑northeast‑3) — **Default site**

### 1) HTTPRoute (attach to existing Gateway; set `X-Site: JP`)

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
    - "api.travels.sandbox802.opentlc.com"
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

### 2) DNSPolicy (JP geo, **defaultGeo: true**)

```bash
oc -n ingress-gateway apply -f - <<'YAML'
apiVersion: kuadrant.io/v1
kind: DNSPolicy
metadata:
  name: prod-web-dnspolicy
  namespace: ingress-gateway
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: prod-web
    sectionName: api
  providerRefs:
    - name: prod-web-aws-credentials
  loadBalancing:
    geo: JP
    defaultGeo: true
    weight: 120
  healthCheck:
    protocol: HTTPS
    port: 443
    interval: 1m
    failureThreshold: 3
    path: /health
YAML
```

---

# Cluster KR (site‑b, ap‑northeast‑2)

### 1) HTTPRoute (attach to existing Gateway; set `X-Site: KR`)

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
    - "api.travels.sandbox802.opentlc.com"
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

### 2) DNSPolicy (KR geo, **defaultGeo: false**)

```bash
oc -n ingress-gateway apply -f - <<'YAML'
apiVersion: kuadrant.io/v1
kind: DNSPolicy
metadata:
  name: prod-web-dnspolicy
  namespace: ingress-gateway
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: prod-web
    sectionName: api
  providerRefs:
    - name: prod-web-aws-credentials
  loadBalancing:
    geo: KR
    defaultGeo: false
    weight: 120
  healthCheck:
    protocol: HTTPS
    port: 443
    interval: 1m
    failureThreshold: 3
    path: /health
YAML
```

---

## Auth — open only the demo endpoints (apply on **both** clusters)

Keep the gateway’s default **deny-all** in place but allow **just** the two paths for the `travels…` host:

```bash
oc -n ingress-gateway apply -f - <<'YAML'
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: geoapi-allow-overrides
  namespace: ingress-gateway
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: prod-web
  overrides:
    when:
      - predicate: "request.host.endsWith('.travels.sandbox802.opentlc.com') && (request.path == '/health' || request.path == '/api/v1/info')"
    rules:
      authentication:
        anon:
          anonymous: {}
YAML
```

> Optional: you can instead attach a route-scoped `AuthPolicy` that allows anonymous, but gateway overrides are the least surprising with multiple apps sharing a listener.


---

## Validate

```bash
# App
oc -n geoapi get deploy/geoapi
oc -n geoapi logs deploy/geoapi | head

# Route status (expect Accepted=True, ResolvedRefs=True; Programmed=True on some versions)
oc -n geoapi get httproute geoapi -o jsonpath='{range .status.parents[*].conditions[*]}{.type}={.status}({.reason}){"\n"}{end}'

# Gateway ELB
oc -n ingress-gateway get gateway prod-web -o jsonpath='{.status.addresses[0].value}{"\n"}'

# Public checks (expect 200 and an X-Site header)
HOST=api.travels.sandbox802.opentlc.com
curl -si https://$HOST/health | head -n1
curl -sI  https://$HOST/api/v1/info | grep -i ^x-site:

# Route53 view (replace ZONE with your hosted zone ID)
ZONE=Z069844912YJ8JXIMRL3Q
aws route53 list-resource-record-sets --hosted-zone-id $ZONE \
  --query "ResourceRecordSets[?Name=='klb.travels.sandbox802.opentlc.com.']" --output table
aws route53 list-resource-record-sets --hosted-zone-id $ZONE \
  --query "ResourceRecordSets[?Name=='jp.klb.travels.sandbox802.opentlc.com.' || Name=='kr.klb.travels.sandbox802.opentlc.com.']" --output table
```

**Force-hit a specific site’s ELB (bypass Route53):**
```bash
ELB=$(oc -n ingress-gateway get gateway prod-web -o jsonpath='{.status.addresses[0].value}')
IP=$(dig +short $ELB | head -1)
HOST=api.travels.sandbox802.opentlc.com
curl -sk --resolve ${HOST}:443:${IP} https://${HOST}/health -o /dev/null -w "%{http_code}\n"
curl -skI --resolve ${HOST}:443:${IP} https://${HOST}/api/v1/info | grep -i ^x-site:
```


---

## Failover drill

**Simulate failure** on one site by breaking `/health` rewrite to `/status/500`:

```bash
oc -n geoapi patch httproute geoapi --type=json -p='[
  {"op":"replace","path":"/spec/rules/1/filters/0/urlRewrite/path/replaceFullPath","value":"/status/500"}
]'
# Watch DNSPolicy enforce removal of that geo
oc -n ingress-gateway describe dnspolicy prod-web-dnspolicy | egrep -A2 'Enforced|SubResourcesHealthy'
```

**Recover**:
```bash
oc -n geoapi patch httproute geoapi --type=json -p='[
  {"op":"replace","path":"/spec/rules/1/filters/0/urlRewrite/path/replaceFullPath","value":"/status/200"}
]'
```

> DNS TTLs: `klb.*` ~**300s**, ELB CNAMEs ~**60s**.


---

## Manual switchover (default site)

```bash
# Make KR the default (run on KR cluster)
oc -n ingress-gateway patch dnspolicy prod-web-dnspolicy --type=merge -p \
  '{"spec":{"loadBalancing":{"defaultGeo":true}}}'

# Make JP the default (run on JP cluster)
oc -n ingress-gateway patch dnspolicy prod-web-dnspolicy --type=merge -p \
  '{"spec":{"loadBalancing":{"defaultGeo":false}}}'
```


---

## Troubleshooting

- **401 Unauthorized** (`www-authenticate: APIKEY`) → another `AuthPolicy` applies. Use the gateway override above.
- **404 Not Found** (`x-envoy-upstream-service-time: 0`) → no route match. Ensure `hostnames` is exact (`api.travels…`) and your routes don’t collide on `prod-web/api`.
  ```bash
  oc get httproute -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.spec.hostnames}{"\t"}{range .spec.parentRefs[*]}{.namespace}/{.name}:{.sectionName}{" "}{end}{"\n"}{end}' \
  | grep 'ingress-gateway/prod-web:api'
  ```
- **SCC/UID errors** → don’t set `runAsUser`; keep `runAsNonRoot: true`, drop all capabilities.
- **No `DNSRecord` CRs** → expected; RHCL writes to Route53 directly. Check `DNSPolicy` for `Enforced=True` and `SubResourcesHealthy=True`.


---

## Cleanup (demo)

```bash
oc -n geoapi delete httproute geoapi || true
oc -n geoapi delete svc geoapi || true
oc -n geoapi delete deploy geoapi || true
oc delete ns geoapi || true
oc -n ingress-gateway delete authpolicy geoapi-allow-overrides || true
# If not managed by Argo:
# oc -n ingress-gateway delete dnspolicy prod-web-dnspolicy || true
```
