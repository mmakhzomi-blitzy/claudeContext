# Helm chart deltas for chat-worker provisioning

## blitzy-client/chart/values.yaml

Add under `images`:

```yaml
images:
  ...
  clientWorker:
    repository: archie-client-worker
    tag: <sha>
  chatWorker:        # NEW — start as a re-tag of clientWorker
    repository: archie-client-worker
    tag: <sha>
```

## blitzy-client/chart/templates/configmap.yaml

Add to the mediator ConfigMap:

```yaml
data:
  ...
  WORKER_IMAGE: "{{ include "chart.image" (dict "Values" .Values "image" .Values.images.clientWorker) }}"
  CHAT_WORKER_IMAGE: "{{ include "chart.image" (dict "Values" .Values "image" .Values.images.chatWorker) }}"
  CHAT_WORKER_TTL_SECONDS: "21600"
  CHAT_WORKER_REAPER_INTERVAL_SECONDS: "300"
  CHAT_WORKER_REAPER_ENABLED: "true"
  CHAT_WORKER_DEFAULT_CPU_REQUEST: "200m"
  CHAT_WORKER_DEFAULT_CPU_LIMIT: "1000m"
  CHAT_WORKER_DEFAULT_MEMORY_REQUEST: "512Mi"
  CHAT_WORKER_DEFAULT_MEMORY_LIMIT: "2Gi"
  CHAT_WORKER_PER_COMPANY_CAP: "50"
```

## blitzy-client/chart/templates/mediator-sa.yaml

Verify the mediator SA's ClusterRole already grants:

```yaml
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "delete", "list", "get", "watch", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

The new chat-runner endpoint relies on `list` with a label selector for the
reaper. If `list` isn't already granted cluster-wide, add it. No new
NetworkPolicy work — chat-worker pods talk to the mediator the same way
existing workers do (Redis queue + WebSocket back-channel).

## helm-chart/apps/archie-service-chat/values.yaml

```yaml
configMaps:
- name: service-chat-config
  data:
    ...
    SERVICE_URL_RELAY: "https://relay.blitzy.dev"     # already needed once we use BlitzyClient
    CHAT_WORKER_PROVISIONING_ENABLED: "false"          # flip per env after smoke
    CHAT_WORKER_TTL_SECONDS: "21600"
    CHAT_WORKER_READY_TIMEOUT: "300"
```

## Per-environment flag flip schedule

| Env | Flip date (target) | Flip method |
|---|---|---|
| dev (mediator dev cluster) | end of week 3 | edit `archie-helm-chart/blitzy-client/chart/values-dev.yaml` |
| qa | week 5 | `values-qa.yaml` |
| stage | week 6 | `values-stage.yaml` (if exists; else qa values are stage's) |
| prod | week 7+ | `values-prod.yaml` (gate on stage burn-in clean for 7 days) |

## Rollback

To disable in any env without code redeploy: edit the relevant
ConfigMap entry `CHAT_WORKER_PROVISIONING_ENABLED=false` and bounce
`archie-service-chat` pods. The chat service falls back to the existing
GCS+Neo4j read path (the FileBackend selector defaults to `Neo4jGcsBackend`
when no `runner_session` is leased).
