# Splunk OpenTelemetry Collector

Configuration for the [Splunk Distribution of the OpenTelemetry Collector](https://github.com/signalfx/splunk-otel-collector),
covering both host/VM deployments and EKS.

## Layout

```
splunk/
├── otel-collector/                     # host and VM deployments (deb/rpm/MSI)
│   ├── agent_config.yaml               # agent mode - upstream default, unmodified
│   ├── gateway_config.yaml             # gateway mode - upstream default, unmodified
│   └── splunk-otel.env.example         # environment the above two configs read
└── kubernetes/                         # EKS deployments (Helm)
    ├── values-eks-agent-only.yaml      # DaemonSet agents, direct export
    ├── values-eks-agent-gateway.yaml   # DaemonSet agents -> gateway pool
    ├── values-eks-fargate.yaml         # Fargate - sidecars + required gateway
    └── manifests/
        ├── namespace.yaml              # apply before helm install
        └── secret.example.yaml         # template for keeping tokens out of values
```

The two files in `otel-collector/` are verbatim copies of the upstream defaults
shipped by the Linux and Windows installer packages. They are checked in as the
baseline to diff against, so local changes stay visible.

## Architecture

Two deployment modes, operating at very different scales.

| | Agent mode | Gateway mode |
|---|---|---|
| Runs on | Every host, VM, or node | A small centralized pool |
| Kubernetes workload | DaemonSet | Deployment (3 replicas by default) |
| Scales with | Node count | Throughput |
| Collects | Host metrics, local app traces and logs | Nothing locally - receives from agents |
| Config | `agent_config.yaml` | `gateway_config.yaml` |

```
Host 1: [agent] ─┐
Host 2: [agent] ─┤
Host 3: [agent] ─┼──►  [gateway pool]  ──►  Splunk Observability Cloud (metrics, traces)
  ...            │      (3 replicas)    └─►  Splunk Platform HEC (logs)
Host N: [agent] ─┘
```

A fleet of 200 hosts runs 200 agents but only 3-5 gateways.

### When to add a gateway

Agent-only is the simpler default and is fine for smaller estates. Add a gateway
tier when you need:

- **Fewer egress points** - allowlist a handful of gateway IPs instead of every host
- **Token isolation** - the access token lives only on the gateway pool
- **Larger buffers and retry queues** - absorbs backend outages better than a per-host buffer
- **Centralized processing** - tail-based sampling needs all spans of a trace to
  converge on one place before a keep/drop decision is possible
- **One place to change processors** - filtering, redaction, and enrichment rules
  updated once rather than across every host

Splunk's guidance puts the crossover at roughly 25+ nodes for Kubernetes.

### Relationship to Splunk Heavy Forwarders

None, for metrics and traces. The OTel Collector belongs to Splunk Observability
Cloud; Heavy Forwarders belong to the Splunk Enterprise/Cloud platform. A gateway
collector is a standalone service, never something installed onto or configured
through a Heavy Forwarder.

The two only meet on the **logs** path. The `splunk_hec` exporter sends to
`SPLUNK_HEC_URL`, which may be Splunk Cloud, an indexer, or a Heavy Forwarder
with HEC enabled:

```
[agent] ──► [gateway] ──► HEC endpoint (may be a Heavy Forwarder) ──► indexers
```

That is the gateway treating the Heavy Forwarder as an ordinary HTTP endpoint,
nothing more.

## Host and VM deployment

Install the collector package, then supply the environment:

```bash
cp otel-collector/splunk-otel.env.example /etc/otel/collector/splunk-otel-collector.conf
# fill in the CHANGEME values
sudo systemctl restart splunk-otel-collector
```

Select the config by pointing `SPLUNK_CONFIG` at `agent_config.yaml` or
`gateway_config.yaml`. Neither file contains a hardcoded URL or credential — every
endpoint and token resolves from the environment.

To route an agent through a gateway rather than direct to Splunk, set
`SPLUNK_GATEWAY_URL` and uncomment the gateway blocks marked
`# Use instead when sending to gateway` in `agent_config.yaml`.

## EKS deployment

Deploy with the official Helm chart. Do not hand-write DaemonSet manifests — the
chart manages the agent DaemonSet, gateway Deployment, and cluster receiver as one
release, and hand-rolled copies drift from it.

```bash
helm repo add splunk-otel-collector-chart \
  https://signalfx.github.io/splunk-otel-collector-chart
helm repo update

kubectl apply -f kubernetes/manifests/namespace.yaml

helm upgrade --install splunk-otel-collector \
  splunk-otel-collector-chart/splunk-otel-collector \
  -n splunk-otel \
  -f kubernetes/values-eks-agent-gateway.yaml
```

Pinned against chart `0.160.0` (collector `0.160.1`).

### Choosing a values file

| Node type | Scale | File |
|---|---|---|
| EC2-backed | under ~25 nodes | `values-eks-agent-only.yaml` |
| EC2-backed | ~25+ nodes, or needs centralized processing | `values-eks-agent-gateway.yaml` |
| Fargate | any | `values-eks-fargate.yaml` |

Node scaling needs no extra work. The DaemonSet controller schedules an agent pod
onto every node that joins and removes it when the node drains — cluster autoscaler
activity is handled natively.

### Fargate

Fargate does not support DaemonSets, so the agent-per-node model does not apply.
The chart runs agent functionality as **sidecars** and the **gateway becomes
mandatory** as the ingestion point. Instrumented applications should report to the
gateway service address:

```
splunk-otel-collector.splunk-otel.svc.cluster.local:4317
```

`clusterName` must be set explicitly on Fargate — there is no EC2 metadata endpoint
for the resource detection processor to derive it from.

### Secrets

The values files carry `accessToken: CHANGEME` inline for readability. For anything
beyond a scratch cluster, create the secret separately and switch the values file to
reference it:

```bash
kubectl create secret generic splunk-otel-collector-tokens \
  -n splunk-otel \
  --from-literal=splunk_observability_access_token='<token>'
```

```yaml
secret:
  create: false
  name: splunk-otel-collector-tokens
```

Key names are fixed by the chart. See `kubernetes/manifests/secret.example.yaml`.

### GitOps

To render manifests for a GitOps pipeline rather than installing directly:

```bash
helm template splunk-otel-collector \
  splunk-otel-collector-chart/splunk-otel-collector \
  -n splunk-otel \
  -f kubernetes/values-eks-agent-gateway.yaml \
  > rendered/splunk-otel-collector.yaml
```

This keeps the chart as the source of truth while still committing manifests.

## Verification

```bash
kubectl get daemonset,deployment -n splunk-otel
kubectl logs -n splunk-otel -l app=splunk-otel-collector --tail=50

# agent pod count should equal node count
kubectl get nodes --no-headers | wc -l
kubectl get pods -n splunk-otel -l component=otel-collector-agent --no-headers | wc -l
```

On hosts, the health check extension answers on port `13133`:

```bash
curl -s localhost:13133
```

## References

- [signalfx/splunk-otel-collector](https://github.com/signalfx/splunk-otel-collector) — collector distribution and default configs
- [signalfx/splunk-otel-collector-chart](https://github.com/signalfx/splunk-otel-collector-chart) — Helm chart
- [Collector deployment modes](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/get-started-understand-and-use-the-collector/deployment-modes)
- [Advanced configuration](https://github.com/signalfx/splunk-otel-collector-chart/blob/main/docs/advanced-configuration.md)
