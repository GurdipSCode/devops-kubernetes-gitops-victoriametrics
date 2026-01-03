# devops-kubernetes-gitops-victoriametrics

Complete monitoring stack for K3s clusters with 100 alert rules and ArgoCD deployment.

## Directory Structure

```
victoria-metrics-k3s/
├── base/
│   ├── namespace.yaml          # victoria-metrics namespace
│   ├── vm-operator.yaml        # VictoriaMetrics Operator ArgoCD App
│   └── vm-stack.yaml           # VM Stack (VMCluster, VMAgent, VMAlert, etc.)
├── alertmanager/
│   └── config.yaml             # Alertmanager configuration
├── alert-rules/
│   ├── 01-node-health.yaml     # Rules 1-15: Node monitoring
│   ├── 02-pod-health.yaml      # Rules 16-35: Pod/container monitoring
│   ├── 03-k3s-components.yaml  # Rules 36-50: K3s/K8s components
│   ├── 04-workloads.yaml       # Rules 51-65: Deployments, StatefulSets, Jobs
│   ├── 05-storage.yaml         # Rules 66-78: PV/PVC, disk I/O
│   ├── 06-networking.yaml      # Rules 79-90: DNS, Traefik, services
│   └── 07-resources.yaml       # Rules 91-100: Quotas, HPA, overcommit
├── kustomization.yaml          # Kustomize for all resources
└── README.md
```

## Quick Start

### Option 1: Apply All Resources

```bash
# Apply everything at once
kubectl apply -k .

# Or apply in order
kubectl apply -f base/namespace.yaml
kubectl apply -f base/vm-operator.yaml
kubectl apply -f base/vm-stack.yaml
kubectl apply -f alertmanager/
kubectl apply -f alert-rules/
```

### Option 2: ArgoCD App of Apps

```bash
# Create ArgoCD application pointing to this repo
argocd app create vm-stack \
  --repo https://github.com/YOUR_ORG/YOUR_REPO.git \
  --path victoria-metrics-k3s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd
```

## Configuration

### 1. Storage Class

Update `storageClassName` in `base/vm-stack.yaml` if not using `local-path`:

```yaml
storageClassName: your-storage-class
```

### 2. Grafana Password

Change the default admin password in `base/vm-stack.yaml`:

```yaml
adminPassword: "your-secure-password"
```

### 3. Alert Notification Sinks

Update `alertmanager/config.yaml` with your notification endpoints. Options include:

**Robusta (if deployed separately):**
```yaml
webhook_configs:
  - url: 'http://robusta-runner.robusta.svc.cluster.local/api/alerts'
```

**Direct Slack:**
```yaml
slack_configs:
  - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
    channel: '#alerts'
```

**PagerDuty:**
```yaml
pagerduty_configs:
  - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
```

## Alert Rules Summary

| File | Rules | Category |
|------|-------|----------|
| 01-node-health.yaml | 1-15 | Node CPU, memory, disk, conditions |
| 02-pod-health.yaml | 16-35 | Pod crashes, OOM, restarts, throttling |
| 03-k3s-components.yaml | 36-50 | API server, kubelet, scheduler, etcd |
| 04-workloads.yaml | 51-65 | Deployments, StatefulSets, DaemonSets, Jobs |
| 05-storage.yaml | 66-78 | PV/PVC status, usage, disk I/O |
| 06-networking.yaml | 79-90 | CoreDNS, Traefik, services, endpoints |
| 07-resources.yaml | 91-100 | Quotas, HPA, cluster overcommit |

## Alert Flow

```
VMAlert (evaluates rules)
    │
    ▼
Alertmanager (routes & groups)
    │
    ▼
Slack / Teams / PagerDuty / Robusta / etc.
```

## Customizing Thresholds

To adjust alert thresholds for different environments, create Kustomize overlays:

```yaml
# overlays/prod/thresholds-patch.yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: k3s-node-health
spec:
  groups:
    - name: k3s-node-health
      rules:
        - alert: NodeHighCPUUsage
          expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
```

## Useful Commands

```bash
# Check VMAlert status
kubectl -n victoria-metrics logs -l app=vmalert

# Check fired alerts in Alertmanager
kubectl -n victoria-metrics port-forward svc/vm-stack-alertmanager 9093:9093
curl localhost:9093/api/v2/alerts

# Query VictoriaMetrics
kubectl -n victoria-metrics port-forward svc/vm-stack-vmselect 8481:8481
curl 'localhost:8481/select/0/prometheus/api/v1/query?query=up'
```

## Troubleshooting

### Alerts not firing
1. Check VMAlert logs: `kubectl -n victoria-metrics logs -l app=vmalert`
2. Verify rules are loaded: Check VMAlert UI or logs for rule count
3. Test PromQL query manually in Grafana

### Alerts not reaching Slack/Teams
1. Check Alertmanager logs: `kubectl -n victoria-metrics logs -l app=alertmanager`
2. Verify webhook URL in alertmanager config

### Missing metrics
1. Verify kube-state-metrics is running
2. Check VMAgent scrape targets
3. Ensure ServiceMonitors are created

## License

MIT
