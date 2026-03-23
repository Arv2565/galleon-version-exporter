cat <<'EOF' > ~/galleon-version-exporter/README.md
# Galleon Version Exporter

A Helm chart that deploys a Prometheus metrics exporter inside each galleon cluster to monitor the version and running status of Kubernetes add-on components. Designed to work with a centralized Grafana dashboard that provides a single pane of glass across all galleon clusters.

## Architecture
```
Galleon Cluster 1                          Galleon Cluster 2
┌─────────────────────────┐                ┌─────────────────────────┐
│  Version Exporter Pod   │                │  Version Exporter Pod   │
│  (queries K8s API)      │                │  (queries K8s API)      │
│         ↓               │                │         ↓               │
│  Local Prometheus       │                │  Local Prometheus       │
│  (scrapes exporter)     │                │  (scrapes exporter)     │
│         │               │                │         │               │
└─────────┼───────────────┘                └─────────┼───────────────┘
          │                                          │
          │         remote-write                     │
          └──────────────┬───────────────────────────┘
                         ↓
              Central Monitoring Cluster
              ┌──────────────────────┐
              │  Central Prometheus  │
              │  (receives metrics)  │
              │         ↓            │
              │  Grafana Dashboard   │
              │  (single pane of     │
              │   glass for all      │
              │   galleons)          │
              └──────────────────────┘
```

## Components Monitored

The exporter tracks 8 add-on components in each galleon cluster:

| Component | Detection Method | Details |
|---|---|---|
| Kubernetes | API `/version` endpoint | Always present, gets `gitVersion` |
| EKS-A | Deployment image tag | Checks `eksa-controller-manager` in `eksa-system` namespace |
| Kubevirt | Custom Resource (CRD) | Reads `status.observedKubeVirtVersion` from KubeVirt CR |
| Rook-Ceph | Deployment image tag | Checks `rook-ceph-operator` in `rook-ceph` namespace |
| GPU Operator | Deployment image tag | Checks `gpu-operator` in `gpu-operator` namespace |
| CDI | Custom Resource (CRD) | Reads `status.observedVersion` from CDI CR |
| ArgoCD | Deployment image tag | Checks `argocd-server` in `argocd` namespace |
| Velero | Deployment image tag | Checks `velero` in `velero` namespace |

## Metrics Exposed(PromQL)

The exporter exposes two metrics per component on port 9100:

| Metric | Labels | Description |
|---|---|---|
| `cluster_addon_info` | `cluster`, `component`, `version` | Version information for each component. Value is always 1. |
| `cluster_addon_status` | `cluster`, `component`, `version` | Running status. `1` = running, `0` = deployed but not running, `-1` = not installed. |

Example output:
```
cluster_addon_info{cluster="houston-galleon",component="kubevirt",version="v1.7.2"} 1
cluster_addon_status{cluster="houston-galleon",component="kubevirt",version="v1.7.2"} 1
cluster_addon_info{cluster="houston-galleon",component="argocd",version="N/A"} 1
cluster_addon_status{cluster="houston-galleon",component="argocd",version="N/A"} -1
```

## Helm Chart Values

| Key | Default | Description |
|---|---|---|
| `galleonName` | `""` | **Required.** Name of the galleon cluster. Used as the `cluster` label in all metrics. |
| `image.repository` | `python` | Container image for the exporter. |
| `image.tag` | `3.12-slim` | Container image tag. |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy. |
| `scrapeInterval` | `300s` | How often Prometheus scrapes the exporter. |

## Setup Guide

### Prerequisites

- Each galleon cluster must have `kube-prometheus-stack` (or equivalent Prometheus Operator setup) installed in the `monitoring` namespace.
- A central monitoring cluster must be available on the same network as all galleon clusters (for remote-write to work).
- `helm` and `kubectl` installed on your machine.

---

### Step 1: Set Up the Central Monitoring Cluster

Deploy `kube-prometheus-stack` on the central cluster with the remote-write receiver enabled so it can accept metrics from all galleon clusters:
```bash
export KUBECONFIG=~/central-cluster.yaml

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install central-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.enableRemoteWriteReceiver=true
```

Wait for all pods to be ready:
```bash
kubectl get pods -n monitoring -w
```

Note the central Prometheus service IP or DNS for later use:
```bash
kubectl get svc -n monitoring | grep prometheus
```

---

### Step 2: Deploy the Version Exporter on Each Galleon

For each galleon cluster, install the Helm chart with the appropriate galleon name:
```bash
export KUBECONFIG=~/galleon-one.yaml

helm install version-exporter ./galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=houston-galleon
```

Repeat for each galleon with a unique `galleonName`:
```bash
export KUBECONFIG=~/galleon-two.yaml

helm install version-exporter ./galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=alaska-edgelight
```

#### Verify the exporter is running:
```bash
kubectl get pods -n version-exporter
```

#### Test the metrics output:
```bash
kubectl port-forward svc/version-exporter-version-exporter 9100:9100 -n version-exporter
# In another terminal:
curl localhost:9100/metrics
```

#### Handle ServiceMonitor label requirements:

Different clusters may have different Prometheus configurations. Check what label selector the local Prometheus expects:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

If it returns something like `{"matchLabels":{"release":"kube-prometheus-stack"}}`, patch the ServiceMonitor to add that label:
```bash
kubectl patch servicemonitor version-exporter-version-exporter -n version-exporter \
  --type merge -p '{"metadata":{"labels":{"release":"kube-prometheus-stack"}}}'
```

If it returns `{}`, no extra label is needed.

#### Verify Prometheus is scraping the exporter:

Port-forward the local Prometheus and check the targets:
```bash
kubectl port-forward svc/<prometheus-svc-name> 9090:9090 -n monitoring
```

Then open `http://localhost:9090/targets` and look for `version-exporter` with state `UP`.

---

### Step 3: Configure Remote-Write from Each Galleon to the Central Prometheus

On each galleon cluster, configure the local Prometheus to push metrics to the central Prometheus. Replace `<CENTRAL_PROMETHEUS_URL>` with the actual IP/hostname of the central cluster's Prometheus service:
```bash
export KUBECONFIG=~/galleon-one.yaml

helm upgrade <existing-prometheus-release-name> prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values \
  --set prometheus.prometheusSpec.remoteWrite[0].url=http://<CENTRAL_PROMETHEUS_URL>:9090/api/v1/write
```

Repeat for each galleon cluster.

To find the existing Prometheus Helm release name:
```bash
helm list -n monitoring
```

To verify remote-write is working, query the central Prometheus for metrics from the galleon:
```bash
# On the central cluster
kubectl port-forward svc/<central-prometheus-svc> 9090:9090 -n monitoring
```

Then query:
```
cluster_addon_info
```

You should see metrics from all configured galleon clusters, each distinguished by the `cluster` label.

---

### Step 4: Set Up the Grafana Dashboard

Access the central Grafana (deployed as part of kube-prometheus-stack on the central cluster):
```bash
# Get the Grafana admin password
kubectl get secret --namespace monitoring central-prom-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Port-forward Grafana
kubectl port-forward svc/central-prom-grafana 3000:80 -n monitoring
```

Open `http://localhost:3000` and log in with `admin` and the password from above.

#### Get the Prometheus datasource UID:
```bash
GRAFANA_PASS="<password-from-above>"

curl -s -u "admin:${GRAFANA_PASS}" http://localhost:3000/api/datasources | python3 -c "
import sys, json
for ds in json.loads(sys.stdin.read()):
    if ds['type'] == 'prometheus':
        print(ds['uid'])
        break
"
```

#### Create the dashboard:

Replace `<DATASOURCE_UID>` with the UID from above:
```bash
cat <<'DASHBOARD' > /tmp/dashboard.json
{
  "dashboard": {
    "title": "Galleon Add-on Versions",
    "uid": "galleon-versions",
    "templating": {
      "list": [
        {
          "name": "cluster",
          "type": "query",
          "datasource": { "type": "prometheus", "uid": "<DATASOURCE_UID>" },
          "query": "label_values(cluster_addon_status{job=~\".*version-exporter.*\"}, cluster)",
          "refresh": 2,
          "includeAll": false,
          "multi": false,
          "current": {}
        }
      ]
    },
    "panels": [
      {
        "id": 1, "title": "Galleon Health", "type": "stat",
        "gridPos": { "h": 4, "w": 24, "x": 0, "y": 0 },
        "datasource": { "type": "prometheus", "uid": "<DATASOURCE_UID>" },
        "targets": [{
          "expr": "count(cluster_addon_status{job=~\".*version-exporter.*\", cluster=\"$cluster\"} == 1) / count(cluster_addon_status{job=~\".*version-exporter.*\", cluster=\"$cluster\"} >= 0) * 100",
          "refId": "A",
          "legendFormat": "$cluster Health"
        }],
        "options": { "textMode": "value_and_name", "colorMode": "background", "graphMode": "none" },
        "fieldConfig": { "defaults": {
          "unit": "percent", "decimals": 0,
          "thresholds": { "mode": "absolute", "steps": [
            { "value": null, "color": "red" },
            { "value": 50, "color": "orange" },
            { "value": 100, "color": "green" }
          ]}
        }}
      },
      {
        "id": 2, "title": "Galleon Add-on Versions", "type": "table",
        "gridPos": { "h": 14, "w": 24, "x": 0, "y": 4 },
        "datasource": { "type": "prometheus", "uid": "<DATASOURCE_UID>" },
        "targets": [
          { "expr": "cluster_addon_info{job=~\".*version-exporter.*\", cluster=\"$cluster\"}", "format": "table", "instant": true, "refId": "A" },
          { "expr": "cluster_addon_status{job=~\".*version-exporter.*\", cluster=\"$cluster\"}", "format": "table", "instant": true, "refId": "B" }
        ],
        "transformations": [
          { "id": "seriesToColumns", "options": { "byField": "component" } },
          { "id": "organize", "options": {
            "excludeByName": {
              "Time 1": true, "Time 2": true, "Value #A": true,
              "cluster 1": true, "cluster 2": true,
              "endpoint 1": true, "endpoint 2": true,
              "instance 1": true, "instance 2": true,
              "job 1": true, "job 2": true,
              "namespace 1": true, "namespace 2": true,
              "pod 1": true, "pod 2": true,
              "service 1": true, "service 2": true,
              "__name__ 1": true, "__name__ 2": true,
              "version 2": true,
              "container 1": true, "container 2": true
            },
            "renameByName": {
              "component": "Component",
              "version 1": "Version",
              "Value #B": "Status"
            }
          }}
        ],
        "fieldConfig": {
          "defaults": { "custom": { "align": "left" } },
          "overrides": [
            { "matcher": { "id": "byName", "options": "Status" }, "properties": [
              { "id": "custom.cellOptions", "value": { "type": "color-background", "mode": "basic" } },
              { "id": "mappings", "value": [
                { "type": "value", "options": { "1": { "text": "RUNNING", "color": "green" } } },
                { "type": "value", "options": { "0": { "text": "NOT RUNNING", "color": "red" } } },
                { "type": "value", "options": { "-1": { "text": "NOT INSTALLED", "color": "dark-grey" } } }
              ]},
              { "id": "custom.width", "value": 200 }
            ]},
            { "matcher": { "id": "byName", "options": "Component" }, "properties": [{ "id": "custom.width", "value": 200 }] }
          ]
        }
      }
    ],
    "schemaVersion": 39,
    "version": 1,
    "refresh": "30s"
  },
  "overwrite": true
}
DASHBOARD

curl -s -u "admin:${GRAFANA_PASS}" \
  -H "Content-Type: application/json" \
  -X POST http://localhost:3000/api/dashboards/db \
  -d @/tmp/dashboard.json
```

The dashboard will be available at `http://localhost:3000/d/galleon-versions` with a dropdown to select individual galleon clusters.

---

## Status Color Codes

| Color | Status Value | Meaning |
|---|---|---|
| Green | `1` | Component is **running** |
| Red | `0` | Component is **deployed but not running** |
| Grey | `-1` | Component is **not installed** on this cluster |

---

## Adding a New Galleon Cluster

1. Install the Helm chart on the new galleon:
```bash
export KUBECONFIG=~/new-galleon.yaml

helm install version-exporter ./galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<new-galleon-name>
```

2. Patch the ServiceMonitor if the local Prometheus requires a specific label:
```bash
kubectl patch servicemonitor version-exporter-version-exporter -n version-exporter \
  --type merge -p '{"metadata":{"labels":{"release":"<prometheus-release-name>"}}}'
```

3. Configure remote-write on the new galleon's Prometheus:
```bash
helm upgrade <prometheus-release> prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values \
  --set prometheus.prometheusSpec.remoteWrite[0].url=http://<CENTRAL_PROMETHEUS_URL>:9090/api/v1/write
```

The new galleon will automatically appear in the Grafana dashboard dropdown.

---

## Adding a New Component to Monitor

Edit the `exporter.py` script in `templates/configmap.yaml` and add a new entry in the `get_all_versions()` function. For example, to add a new component called `istio`:

1. Add it to the default results dictionary:
```python
results = {
    ...
    "istio": {"version": "N/A", "status": -1},
}
```

2. Add the detection logic (using deployment image tag method):
```python
ver, status = get_deployment_version("istio-system", "istiod")
if ver:
    results["istio"] = {"version": ver, "status": status}
```

3. Upgrade the Helm chart on all galleon clusters:
```bash
helm upgrade version-exporter ./galleon-version-exporter/ \
  --namespace version-exporter \
  --reuse-values
```

---

## Troubleshooting

### Exporter shows all N/A

Check the exporter logs:
```bash
kubectl logs -n version-exporter deployment/version-exporter-version-exporter
```

Verify the ServiceAccount has the correct RBAC permissions:
```bash
kubectl get clusterrole version-exporter-version-exporter -o yaml
```

### ServiceMonitor not picked up by Prometheus

Check what label selector Prometheus expects:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

Ensure the ServiceMonitor has the matching label.

### Remote-write not working

Verify the central Prometheus has the receiver enabled:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.enableRemoteWriteReceiver}' ; echo
```

Check the galleon Prometheus logs for remote-write errors:
```bash
kubectl logs -n monitoring statefulset/prometheus-<release>-prometheus -c prometheus | grep remote
```

Verify network connectivity from the galleon to the central Prometheus:
```bash
kubectl run test-net --rm -it --image=busybox --restart=Never -- wget -qO- http://<CENTRAL_PROMETHEUS_URL>:9090/api/v1/status/config
```
EOF

echo "README created at ~/galleon-version-exporter/README.md"
