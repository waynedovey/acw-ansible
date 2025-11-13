# RHCL (Kuadrant) Geo DNS + Existing Gateway — JSON API Demo (PodSecurity‑safe)

This guide sets up **Geo DNS** across two OpenShift clusters using **Red Hat Connectivity Link (RHCL/Kuadrant)**, **Gateway API**, and your **existing Gateway**:

- Clusters: **JP** (ap-northeast-3, “site‑c”) and **KR** (ap-northeast-2, “site‑b”)
- Gateway: `ingress-gateway/prod-web` (listener `api`)
- Domain: `api.travels.sandbox802.opentlc.com` (TLS via `api-tls`)
- Demo App: `geoapi` with `/api/v1/info` and `/health`
- Image: **public** `quay.io/wdovey/go-httpbin:latest` (runs on **8080**, PodSecurity “restricted” compatible)
- Geo routing & health‑checked failover managed by **DNSPolicy**

> We use a **single exact host** (`api.travels.sandbox802.opentlc.com`) to avoid wildcard route collisions with other demo apps on the same Gateway.


---

## Prerequisites

- RHCL (Kuadrant) installed on both clusters
- Gateway API v1 CRDs and a working `GatewayClass` (e.g., `istio`)
- Existing `Gateway` **prod-web** in namespace **ingress-gateway** with listener **api** and a certificate secret `api-tls` covering `*.travels.sandbox802.opentlc.com`
- Route53 hosted zone `travels.sandbox802.opentlc.com` and AWS credentials with access to manage it
- Credentials secret in **each** cluster/namespace:
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
    # AWS_REGION: <base64 of ap-northeast-2>
  ```

---

## 1) Deploy the demo app (both clusters)

```bash
# Namespace + Deployment + Service (PodSecurity safe, listens on :8080)
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

## 2) HTTPRoute (attach to existing Gateway)

- Hostname is **exact**: `api.travels.sandbox802.opentlc.com`
- `/api/v1/info` → rewrites to `/headers` and sets `X-Site` header to **JP** or **KR**
- `/health` → rewrites to `/status/200` (used by DNS health checks)

### Apply on **JP (site‑c, ap‑northeast‑3)** — set `X-Site: JP`
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

### Apply on **KR (site‑b, ap‑northeast‑2)** — set `X-Site: KR`
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

> If you share the `prod-web/api` listener with other apps, **avoid wildcards** on those routes or use distinct hostnames to prevent 404s due to precedence.


---

## 3) DNSPolicy (Geo + health‑checked failover)

You can deploy via your **Ansible** playbook (recommended) or apply YAML directly.

### Ansible (per cluster)
> Don’t quote booleans.

**JP (site‑c, default = true):**
```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=JP \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=true \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120
```

**KR (site‑b):**
```bash
ansible-playbook playbooks/ocp4_workload_app_connectivity_workshop.yml \
  -e ACTION=create \
  -e ocp4_workload_app_connectivity_workshop_acme_email=wdovey@gmail.com \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_geo_code=KR \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_default=false \
  -e ocp4_workload_app_connectivity_workshop_dnspolicy_weight=120
```

### (Alt) YAML (adjust `geo`/`defaultGeo` per cluster)
```yaml
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
    # sectionName: api   # optional if policy is for the whole gateway
  providerRefs:
    - name: prod-web-aws-credentials
  loadBalancing:
    geo: JP|KR         # set per site
    defaultGeo: true|false
    weight: 120
  healthCheck:
    protocol: HTTPS
    port: 443
    interval: 1m
    failureThreshold: 3
    path: /health
```

**Resulting DNS shape:**
```
*.travels…     → CNAME klb.travels…
klb.travels…   → CNAME {JP|KR|default} → {jp|kr}.klb.travels…
{jp|kr}.klb…   → CNAME <site ELB>
```
RHCL keeps Route53’s single‑value CNAME requirement while enabling GEO + auto‑failover.


---

## 4) Auth — open ONLY the demo endpoints

Keep your Gateway’s default **deny‑all** (`prod-web-deny-all`) for safety and allow just the **two demo paths** for this host.

```bash
# Apply on both clusters
oc apply -f - <<'YAML'
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

> If you prefer route‑scoped policy instead, use:
> ```yaml
> apiVersion: kuadrant.io/v1
> kind: AuthPolicy
> metadata: { name: geoapi-open, namespace: geoapi }
> spec:
>   targetRef:
>     group: gateway.networking.k8s.io
>     kind: HTTPRoute
>     name: geoapi
>   rules:
>     authentication:
>       allow-anon:
>         anonymous: {}
> ```
> Note: a Gateway policy with `overrides` can trump route policies; the gateway override above is the simplest, least‑surprising option.


---

## 5) Validate

```bash
# App
oc -n geoapi get deploy/geoapi
oc -n geoapi logs deploy/geoapi | head

# Route (expect Accepted=True, Programmed=True)
oc -n geoapi get httproute geoapi -o jsonpath='{range .status.parents[*].conditions[*]}{.type}={.status}({.reason}){"\n"}{end}'

# Gateway ELB
oc -n ingress-gateway get gateway prod-web -o jsonpath='{.status.addresses[0].value}{"\n"}'

# Public checks (expect 200 and an X-Site header)
HOST=api.travels.sandbox802.opentlc.com
curl -si https://$HOST/health | head -n1
curl -sI  https://$HOST/api/v1/info | grep -i ^x-site:

# Route53 view
ZONE=<YOUR_HOSTED_ZONE_ID>   # e.g. Z069844912YJ8JXIMRL3Q
aws route53 list-resource-record-sets --hosted-zone-id $ZONE \
  --query "ResourceRecordSets[?Name=='klb.travels.sandbox802.opentlc.com.']" --output table
aws route53 list-resource-record-sets --hosted-zone-id $ZONE \
  --query "ResourceRecordSets[?Name=='jp.klb.travels.sandbox802.opentlc.com.' || Name=='kr.klb.travels.sandbox802.opentlc.com.']" --output table
```

**Force hitting a specific site’s ELB (bypass Route53):**
```bash
ELB=$(oc -n ingress-gateway get gateway prod-web -o jsonpath='{.status.addresses[0].value}')
IP=$(dig +short $ELB | head -1)
HOST=api.travels.sandbox802.opentlc.com
curl -sk --resolve ${HOST}:443:${IP} https://${HOST}/health -o /dev/null -w "%{http_code}\n"
curl -skI --resolve ${HOST}:443:${IP} https://${HOST}/api/v1/info | grep -i ^x-site:
```


---

## 6) Failover drill

**Simulate failure** on one site by breaking `/health` rewrite to `/status/500`:

```bash
oc -n geoapi patch httproute geoapi --type=json -p='[
  {"op":"replace","path":"/spec/rules/1/filters/0/urlRewrite/path/replaceFullPath","value":"/status/500"}
]'
# Watch DNSPolicy until it re-enforces, removing this geo from klb.*
oc -n ingress-gateway describe dnspolicy prod-web-dnspolicy | egrep -A2 'Enforced|SubResourcesHealthy'
```

**Recover**:
```bash
oc -n geoapi patch httproute geoapi --type=json -p='[
  {"op":"replace","path":"/spec/rules/1/filters/0/urlRewrite/path/replaceFullPath","value":"/status/200"}
]'
```

> Remember Route53 TTLs: `klb.*` is typically **300s**, ELB CNAMEs ~60s.


---

## 7) Manual switchover (default site)

Flip which geo answers **non‑geo** queries:

```bash
# Make KR the default
oc -n ingress-gateway patch dnspolicy prod-web-dnspolicy --type=merge -p \
  '{"spec":{"loadBalancing":{"defaultGeo":true}}}'

# Make JP the default
oc -n ingress-gateway patch dnspolicy prod-web-dnspolicy --type=merge -p \
  '{"spec":{"loadBalancing":{"defaultGeo":false}}}'
```


---

## 8) Troubleshooting

- **401 Unauthorized** with `www-authenticate: APIKEY` → an Authorino AuthPolicy is in effect. Use the gateway override above, or a route‑scoped allow‑anon.
- **404 Not Found** from `istio-envoy` with `x-envoy-upstream-service-time: 0` → **routing mismatch** (no route matched). Ensure your HTTPRoute `hostnames` is **exact** and unique on the shared `prod-web/api` listener.
  ```bash
  oc get httproute -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.spec.hostnames}{"\t"}{range .spec.parentRefs[*]}{.namespace}/{.name}:{.sectionName}{" "}{end}{"\n"}{end}' \
  | grep 'ingress-gateway/prod-web:api'
  ```
- **App port conflicts** → keep container on **8080**, Service on **80**; don’t bind to 80 in the container.
- **SCC/UID errors** → don’t set `runAsUser`; keep `runAsNonRoot: true`, drop all capabilities.
- **No DNSRecord CRs** → RHCL manages Route53 directly (you’ll see TXT ownership markers); check `DNSPolicy` status for `Enforced=True` and `SubResourcesHealthy=True`.


---

## 9) Clean up (demo only)

```bash
oc -n geoapi delete httproute geoapi || true
oc -n geoapi delete svc geoapi || true
oc -n geoapi delete deploy geoapi || true
oc delete ns geoapi || true
# Auth and DNS policies (if you created them via YAML; keep if managed by Argo)
oc -n ingress-gateway delete authpolicy geoapi-allow-overrides || true
# oc -n geoapi delete authpolicy geoapi-open || true
# oc -n ingress-gateway delete dnspolicy prod-web-dnspolicy || true
```
