# Customer Install — Debug Notes

Captured from cluster bring-up sessions on the **GHES customer cluster** and **`cicd-blitzy` cluster** during late April / early May 2026. Each entry: symptoms, root cause, fix.

---

## 1. CNPG operator stuck on startup — `forbidden` errors

### Symptoms
Operator pod logs show a sequence of `forbidden` errors as it attempts each startup step:
```
unable to read ConfigMap ... configmaps "cnpg-controller-manager-config" is forbidden
unable to setup PKI infrastructure ... deployments.apps is forbidden
mutatingwebhookconfigurations.admissionregistration.k8s.io "cnpg-mutating-webhook-configuration" is forbidden
```
The trailing `Azure does not have opinion for this user.` is AKS Azure-RBAC noise — actual denial is plain K8s RBAC.

### Root cause
**Two independent bugs depending on which cluster:**

1. **GHES cluster**: ClusterRoleBinding `blitzy-client-cloudnative-pg` had `subjects[0].namespace: default`, but the operator SA was in `blitzy-client`. ClusterRole was correct; binding pointed at a non-existent SA. **Chart bug** — likely `subjects[0].namespace` was hardcoded `default` instead of `{{ .Release.Namespace }}` somewhere upstream.

2. **`cicd-blitzy` cluster**: The operator's namespace-scoped Role didn't exist at all. The chart's RBAC manifests were getting blocked by Argo's privilege-escalation guard (Argo's controller SA didn't have `escalate`/`bind` on `roles`).

### Fix
For (1):
```bash
kubectl patch clusterrolebinding blitzy-client-cloudnative-pg \
  --type=json \
  -p='[{"op":"replace","path":"/subjects/0/namespace","value":"blitzy-client"}]'
```

For (2): manually applied a comprehensive Role + RoleBinding to the operator namespace. Saved at `/Users/m.makhzomi/Downloads/cnpg-operator-role.yaml`. Final form ends up wildcarded (`apiGroups: ["*"]`, `resources: ["*"]`, `verbs: ["*"]`) for the namespace Role + ClusterRole for cluster-scoped resources (webhook configs, CRDs, nodes).

### Related quirks to know
- `resourceNames: ["*"]` in a Role rule does **not** wildcard — it's exact-string match for a resource literally named `*`. Use `resources: ["*"]` instead, or omit `resourceNames` entirely.
- CNPG creates per-`Cluster` Role/RoleBinding **at runtime** (owner-ref to the `Cluster` CR). Don't write these by hand — operator overwrites/garbage-collects them.
- The operator does **not** use a Job to create `<cluster>-app` Secret. Secret is created by operator's reconcile loop before instance Pod starts.

---

## 2. Vault HA TLS deadlock under Argo CD

### Symptoms
Vault StatefulSet pod stuck `ContainerCreating` with `FailedMount`:
```
MountVolume.SetUp failed for volume "userconfig-vault-ha-tls" :
secret "vault-ha-tls" not found
```

### Root cause
Architectural mismatch between Helm and Argo:
- Chart's `post-install.yaml` is a `helm.sh/hook: post-install` Job → Argo translates to `argocd.argoproj.io/hook: PostSync`.
- Vault StatefulSet is a regular Sync-phase resource.
- Argo waits for **all Sync-phase resources to be Healthy** before running PostSync hooks.
- Vault StatefulSet can't become Healthy because it's blocked on the Secret.
- Secret would have been created by `post-install.sh` (PostSync) → deadlock.

(Helm directly doesn't have this problem — `helm install` runs hooks immediately. Argo's stricter phase-gating exposes the cycle.)

### Fix
Manual out-of-band Secret bootstrap before/during first Argo sync. The `generate_certificate()` function in `chart/scripts/post-install.sh` is the reference implementation — it uses K8s CSR + cluster CA to sign. Steps:
1. `openssl genrsa` → key
2. `openssl req` → CSR with SANs for `*.<release>-vault-internal`, `*.<release>-vault-internal.<ns>.svc.cluster.local`, `*.<ns>`, `127.0.0.1`
3. Submit `CertificateSigningRequest` resource (signer `kubernetes.io/kubelet-serving` on most clusters; `beta.eks.amazonaws.com/app-serving` on EKS)
4. `kubectl certificate approve vault.svc`
5. Read `.status.certificate`, write key/crt/ca to a Secret named `vault-ha-tls`

### Long-term fix (not done)
Restructure chart so the Secret is created in Sync phase, not PostSync. Or use cert-manager's `Certificate` CR to create the Secret declaratively (the script already has a `certificate_manager()` codepath for this when cert-manager is installed first).

---

## 3. Vault unseal — secondary pods stranded

### Symptoms
After re-running `post-install.sh`, primary pod (`vault-0`) is unsealed but `vault-1`/`vault-2` log:
```
seal configuration missing. Not initialized, security barrier not initialized
```

### Root cause
Secondary pods have empty raft storage — never joined the cluster. The `enable_vault()` function's raft-join step failed mid-loop, leaving them stranded.

### Fix (manual recovery)
For each non-leader pod:
```bash
NS=cicd-blitzy
RELEASE=cicd-blitzy
VAULT_SVC="${RELEASE}-vault-internal"
MAIN_POD="${RELEASE}-vault-0"

kubectl exec -n "$NS" "$p" -- sh -c "
  VAULT_ADDR=https://127.0.0.1:8200 \
  VAULT_CACERT=/vault/userconfig/vault-ha-tls/vault.ca \
  vault operator raft join \
    -address=https://$p.$VAULT_SVC:8200 \
    -leader-ca-cert=\"\$(cat /vault/userconfig/vault-ha-tls/vault.ca)\" \
    -leader-client-cert=\"\$(cat /vault/userconfig/vault-ha-tls/vault.crt)\" \
    -leader-client-key=\"\$(cat /vault/userconfig/vault-ha-tls/vault.key)\" \
    https://$MAIN_POD.$VAULT_SVC:8200"

# Then unseal with threshold keys from vault-seal Secret
KEYS=($(kubectl get secret vault-seal -n "$NS" \
  -o jsonpath='{.data.cluster-keys\.json}' | base64 -d | jq -r '.unseal_keys_b64[]'))
for ((i=0; i<3; i++)); do
  kubectl exec -n "$NS" "$p" -- sh -c \
    "VAULT_ADDR=https://127.0.0.1:8200 VAULT_CACERT=/vault/userconfig/vault-ha-tls/vault.ca \
     vault operator unseal '${KEYS[$i]}'"
done
```

### Related: stale raft peers
If a peer's PVC was wiped and the pod rescheduled, the raft cluster may still list the old peer. Symptom: `raft join` succeeds but pod stays uninitialized.
```bash
# On leader, after vault login with root token:
vault operator raft list-peers
vault operator raft remove-peer <node_id>
# Then re-join from the affected pod
```

---

## 4. Mediator stuck on "No master found for 'mymaster'"

### Symptoms
```
Outbound drain skipped, Redis unavailable at queue-length check:
No master found for 'mymaster' : <redis... port=26379...> - network:TimeoutError
```
Mediator otherwise functional (RQ worker connects, `Redis is reachable` logs appear).

### Root cause
`blitzy-config` ConfigMap had `REDIS_SENTINEL_HOST=blitzy-redis...:26379` set, but **the cluster had no Sentinel running**. STS `blitzy-redis` had only the `redis:7.2` container; no sentinel sidecar, no Sentinel pod, no port 26379 anywhere. Mediator's `redis_client_provider.py` chose Sentinel mode because the env var was set, hit an empty TCP port, timed out.

The chart values had:
```yaml
redis:
  sentinel:
    masterSet: mymaster        # ← set
                               # ← missing: enabled: true
```
Without `enabled: true` on the bitnami subchart, sentinel resources were never templated.

### Fix
Removed sentinel entirely from both code and chart (decision: simpler than fighting bitnami's 2026 image-publishing changes).

**Mediator code:**
- `src/utils/redis_client_provider.py` — dropped Sentinel branch
- `src/consts.py` — removed `REDIS_SENTINEL_*` constants
- `main.py` — dropped `REDIS_SENTINEL_HOST` import + branch

**Chart:**
- `templates/redis/statefulset.yaml` — sentinel sidecar block kept conditional but `redis.sentinel:` block removed from values.yaml so it doesn't render
- `templates/redis/service.yaml` — same pattern
- `templates/configmap.yaml` — `REDIS_SENTINEL_*` keys removed
- `templates/_helpers.tpl` — `redisSentinelHost`/`redisSentinelPort` helpers removed

To re-enable sentinel later: add `redis.sentinel.enabled: true` + the rest of the block back to `values.yaml`. The conditional `{{- if .Values.redis.sentinel.enabled }}` blocks in STS/Service templates will render automatically.

### Live-cluster patches we made (still active until Argo re-syncs)
- Added sentinel sidecar container to `blitzy-redis` STS via `kubectl patch`
- Added port 26379 to `blitzy-redis` Service
- Removed `REDIS_SENTINEL_*` from `blitzy-config` (then verified config worked direct-mode)

These will be reverted when chart is re-applied. Cleanup commands in the chat history if needed before that.

---

## 5. Bitnami sentinel image — `8.6.0` doesn't exist publicly

### Discovery
While trying to mirror `bitnami/redis-sentinel:8.6.0` to GCP AR, found:
- `docker.io/bitnami/redis-sentinel:8.6.0` → not found
- `docker.io/bitnami/redis-sentinel:7.4.6, 7.4.5, 7.4.0, ...` → not found
- `docker.io/bitnamilegacy/redis-sentinel:latest` → exists (only this tag)

### Cause
Bitnami changed their hosting model in late 2025. Free version-tagged images are no longer published. Free tier kept only `:latest` on `bitnamilegacy/`. Versioned images moved to paid `bitnamicharts/` / `bitnamipremier/`.

### Workaround
Use the official `redis:7.2` image (or any tag) for sentinel — it ships both binaries:
- `/usr/local/bin/redis-server`
- `/usr/local/bin/redis-sentinel`

Single image, one mirror entry, simpler. This is what the live-cluster sentinel sidecar uses and what we verified working.

### Found later
A `redis-sentinel:8.6.0` image **does** exist at `us-docker.pkg.dev/blitzy-shared/infra-images/redis-sentinel:8.6.0` (someone mirrored it from somewhere — not Docker Hub). It's a bitnami-style image; works with our chart's `command:` block since `redis-sentinel` is in PATH at `/opt/bitnami/redis-sentinel/bin/`.

---

## 6. AR image mirroring — preserve multi-arch

### Symptom
After `docker pull` + `docker tag` + `docker push`, the mirrored image only had one platform manifest (whichever the pulling machine's arch was — arm64 on Mac). Customer's amd64 nodes would fail to schedule.

### Cause
`docker pull <multi-arch-tag>` pulls only the host's architecture. Subsequent `push` only sends that single manifest, not the original multi-arch index.

### Fix
Use `docker buildx imagetools create` to copy the manifest list directly server-side:
```bash
docker buildx imagetools create \
  --tag us-docker.pkg.dev/blitzy-shared/infra-images/<repo>:<tag> \
  docker.io/<source>:<tag>
```
This preserves all platforms and digests exactly. Layers are referenced, not re-uploaded.

### Linux-only filter
Bitnami/Docker Hub multi-arch indexes often include `unknown/unknown` manifests (SLSA attestations) that confuse some K8s scanners. Strip them by listing only the linux platforms in `imagetools create`:
```bash
docker buildx imagetools create \
  --tag us-docker.pkg.dev/blitzy-shared/<repo>:<tag> \
  docker.io/<src>:<tag>@sha256:<linux-amd64-digest> \
  docker.io/<src>:<tag>@sha256:<linux-arm64-digest>
```

### Path-preserving vs flat
Two patterns we use:
- **Flat**: `docker.io/otel/opentelemetry-collector-k8s:0.147.0` → `infra-images/otel-collector-k8s:0.147.0`
- **Path-preserving**: `docker.io/otel/opentelemetry-collector-k8s:0.146.1` → `infra-images/otel/opentelemetry-collector-k8s:0.146.1`

Mismatch between chart's `repository:` value and what's actually mirrored is a common source of "manifest not found" errors. Always confirm by `gcloud artifacts docker tags list <full-path>`.

---

## 7. CRDs — fresh-install comparison

### Use case
Before running `helm install` / Argo sync on a new cluster, confirm whether prerequisite CRDs are present.

### Command
```bash
comm -23 <(cat <<'EOF' | sort
certificates.cert-manager.io
certificaterequests.cert-manager.io
issuers.cert-manager.io
clusterissuers.cert-manager.io
challenges.acme.cert-manager.io
orders.acme.cert-manager.io
clusters.postgresql.cnpg.io
backups.postgresql.cnpg.io
scheduledbackups.postgresql.cnpg.io
poolers.postgresql.cnpg.io
imagecatalogs.postgresql.cnpg.io
clusterimagecatalogs.postgresql.cnpg.io
publications.postgresql.cnpg.io
subscriptions.postgresql.cnpg.io
databases.postgresql.cnpg.io
failoverquorums.postgresql.cnpg.io
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
tlsroutes.gateway.networking.k8s.io
tcproutes.gateway.networking.k8s.io
udproutes.gateway.networking.k8s.io
grpcroutes.gateway.networking.k8s.io
referencegrants.gateway.networking.k8s.io
envoyproxies.gateway.envoyproxy.io
envoypatchpolicies.gateway.envoyproxy.io
clienttrafficpolicies.gateway.envoyproxy.io
backendtrafficpolicies.gateway.envoyproxy.io
securitypolicies.gateway.envoyproxy.io
envoyextensionpolicies.gateway.envoyproxy.io
backends.gateway.envoyproxy.io
EOF
) <(kubectl get crd -o name | sed 's|customresourcedefinition.apiextensions.k8s.io/||' | sort)
```
Prints CRDs missing from the cluster. Empty output = all 31 prerequisites present.

---

## 8. Helm gotchas observed

### `enabled: 'false'` is a string
```yaml
cert-manager:
  enabled: 'false'   # ← string, evaluates TRUTHY in {{ if }} → resources STILL render
certificate:
  enabled: false     # ← boolean, evaluates falsy → skipped
```
Always omit quotes for boolean fields.

### Subcharts in `charts/` directory render even without `Chart.yaml` deps
Helm auto-discovers `.tgz` files in `charts/`. Removing a dep from `Chart.yaml` does **not** stop the subchart from rendering if its archive is still in `charts/`. Delete the `.tgz` to fully disable.

The published 1.0.34 chart has no `charts/` directory — bitnami subcharts were dropped. The local working copy at 1.0.29 still had them, which is why both bitnami and custom redis resources were appearing in renders.

### Argo sync-wave only orders within phase
`argocd.argoproj.io/sync-wave: "-10"` does **not** make a Sync resource run before a PreSync hook. PreSync always runs first. To affect cross-phase ordering, change the phase (`hook: PreSync` vs `hook: Sync`).

---

## 9. Mediator gotchas observed (from prior sessions, kept here for reference)

### `decode_responses=True` + RQ → UnicodeDecodeError
Setting `decode_responses=True` on the redis client used by RQ workers crashes with:
```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x9c
```
RQ stores pickled bytes in Redis; auto-decoding mangles them. Keep `decode_responses=False` for any RQ-related connection. Local dev mode is fine because RQ isn't used there.

### `archive_by_server_id` `NotNullViolation` on `is_deleted`
The `command_executions` table inherits `VisibilityMixin` (`is_deleted`, `deleted_at`). The `INSERT INTO command_execution_archive ... SELECT ...` must include those columns explicitly — `false`, `null` literal works.

### Synthetic `Response` for SDK errors had `url=None`, `reason=None`
`requests.models.Response()` with no init has all fields blank. When raising in `_poll_until_complete`, propagate the original URL/reason or build a clearer error message.

---

## 10. Chart LB topology (for reference)

| Path | Trigger | Resource |
|---|---|---|
| `gateway.enabled: false` | Default | `templates/client-mediator.yaml` Service `type: LoadBalancer` named `<release>-client-mediator-lb` |
| `gateway.enabled: true` | Explicit | `templates/gateway.yaml` Gateway CR → Envoy Gateway controller provisions Service of `type: LoadBalancer` |

Customer egress requirements:
- Relay accepts both **HTTP (80)** and **HTTPS (443)**, no auto-redirect
- `MAIN_SERVER_URL=https://platform.api-k.blitzy.com` (default)
- HTTP-only egress works if customer overrides `MAIN_SERVER_URL` to `http://...`

---

## 11. Vault stays sealed after restart — `vault-unseal` CronJob hits 403 on Secret

### Symptoms
- After a Vault pod restart, `vault-0` shows `Sealed: true` and stays that way.
- Mediator (and any other consumer reading from Vault) fails to fetch secrets.
- `vault-unseal` CronJob pod log shows:
  ```
  Error from server (Forbidden): secrets "vault-seal" is forbidden:
    User "system:serviceaccount:<ns>:<release>-service-account" cannot get
    resource "secrets" in API group "" in the namespace "<ns>"
  ```
- The CronJob keeps failing every 30 minutes; Vault never auto-unseals.

### Root cause
`templates/service-account.yaml` defines `<release>-role` (bound to `<release>-service-account`) with permissions on `pods`, `pods/exec`, `configmaps`, `deployments`, `deployments/scale` — but **not `secrets`**. The `vault-unseal.yaml` CronJob uses this SA to read the `vault-seal` Secret containing unseal keys. With no `secrets:get`, the read fails 403, Vault stays sealed.

This is silent at deploy time — K8s only surfaces the failure at runtime when the CronJob actually fires, which is also the moment Vault becomes unavailable.

### Fix (chart side — durable)

Add a `secrets` rule to `<release>-role` in `templates/service-account.yaml`:
```yaml
rules:
  # ... existing rules ...
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  # ... existing rules ...
```

For tighter blast radius, scope by resource name:
```yaml
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["vault-seal"]
    verbs: ["get"]
```

After the chart fix lands and the next ArgoCD sync runs, future Vault restarts auto-unseal within 30 minutes (CronJob schedule).

### Hotfix (live cluster — until chart lands)

Two manual commands — see `roughwork/vault_rbac_fix.md` for the runbook:

```bash
# 1. Patch the Role in-place
kubectl patch role <release>-role -n <ns> --type=json -p='[
  {"op":"add","path":"/rules/-","value":{"apiGroups":[""],"resources":["secrets"],"verbs":["get","list","watch"]}}
]'

# 2. Unseal vault-0 directly using your own kubectl creds (bypasses the SA)
NS=<ns>
POD=<release>-vault-0
KEYS=$(kubectl get secret vault-seal -n $NS \
  -o jsonpath='{.data.cluster-keys\.json}' | base64 -d \
  | python3 -c "import json,sys; print('\n'.join(json.load(sys.stdin)['unseal_keys_b64']))")
COUNT=0
for KEY in $KEYS; do
  kubectl exec -n $NS $POD -- sh -c \
    "VAULT_ADDR=https://127.0.0.1:8200 vault operator unseal \"$KEY\""
  COUNT=$((COUNT + 1))
  [ $COUNT -ge 3 ] && break
done
```

### ArgoCD note
A live `kubectl patch role` will be reverted on the next ArgoCD sync unless:
- The chart fix lands and the Application is pointed at the new chart version, **before** the next sync; OR
- The Application has `ignoreDifferences` configured for `Role.rules`; OR
- `selfHeal: false` is set temporarily.

### Related — chart secret format (for the unseal script)
The `vault-seal` Secret has a single key `cluster-keys.json` whose decoded value is a JSON blob:
```json
{
  "unseal_keys_b64": ["<key1>", "<key2>", "<key3>", "<key4>", "<key5>"],
  "unseal_keys_hex": [...],
  "unseal_shares": 5,
  "unseal_threshold": 3,
  "recovery_keys_b64": [...],
  "root_token": "hvs.xxxxx"
}
```
Vault's endpoint is `https://127.0.0.1:8200` (TLS, not plain HTTP) — set `VAULT_ADDR=https://127.0.0.1:8200` when execing.

### Related — cosmetic `service_registration.kubernetes` 403 warning
Vault's pod log will continue to show:
```
service_registration.kubernetes: unable to set initial state due to PATCH ... 403
```
even after the unseal fix. This is a **separate** RBAC gap — Vault's *own* SA (`<release>-vault-service-account`, used by the StatefulSet) lacks `patch` on `pods`, so Vault can't self-label `vault-active=true`. Cosmetic, Vault runs fine without it. Fix in `templates/vault/service-account.yaml` ClusterRole by adding `patch` to the pods/nodes verbs:

```yaml
  - apiGroups: [""]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch", "patch"]   # ← add patch
```
