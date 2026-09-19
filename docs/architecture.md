# Architecture

## High-level design

```text
                   +-----------------------------+
                   |     Kubernetes Cluster      |
                   |                             |
                   |  sas1   sas2   sas3         |
                   |    \      |      /          |
                   |     \     |     /           |
                   |      Falco DaemonSet         |
                   |       on every node          |
                   +-------------|---------------+
                                 |
                                 | modern eBPF
                                 v
                      +----------------------+
                      | Falco rule engine    |
                      | built-in + custom    |
                      +----------+-----------+
                                 |
                    +------------+-------------+
                    |                          |
                    v                          v
            Falco alert logs            Falco native metrics
                                               |
                                               v
                                    Service: falco-metrics
                                        TCP 8765 /metrics
                                               |
                                               v
                                        ServiceMonitor
                                               |
                                               v
                                 kube-prometheus-stack
                                               |
                                               v
                                            Grafana
```

## Node-level placement

Falco runs as a DaemonSet so every Kubernetes node has a local runtime sensor:

- `dhn-ceph-sas1`
- `dhn-ceph-sas2`
- `dhn-ceph-sas3`

## Runtime event path

1. A process inside a container performs an action.
2. The Linux kernel generates the corresponding event.
3. Falco's modern eBPF driver captures it.
4. The Falco rule engine evaluates built-in and custom rules.
5. Matching rules generate alert output.
6. Rule counters are exposed through the native Prometheus endpoint.

## Metrics path

```text
Falco pod
  |
  v
:8765/metrics
  |
  v
falco-metrics Service
  |
  v
ServiceMonitor
  |
  v
Prometheus Operator
  |
  v
Prometheus TSDB
  |
  v
Grafana
```

## Why the existing Prometheus stack was reused

The cluster already had `kube-prometheus-stack`. Reusing it avoided duplicate Prometheus/Grafana installations, wasted resources, extra datasources, and unnecessary operational complexity.

The Falco ServiceMonitor carries:

```yaml
release: kube-prometheus-stack
```

because the existing Prometheus instance selects ServiceMonitors using that label.

## Why `falco-exporter` was not used

`falco-exporter` was investigated but found to be deprecated. Falco 0.38+ exposes Prometheus metrics directly, and this project uses Falco 0.44.1.

Final design:

```text
Falco -> native /metrics -> Prometheus
```

instead of:

```text
Falco -> gRPC -> falco-exporter -> Prometheus
```

## Harbor path

```text
Falco Helm release
      |
      v
imagePullSecrets: harbor-secret
      |
      v
Dedicated pull-only robot account
      |
      v
repo.fiberathome.cloud/falco/falco:0.44.1
```

No registry password or robot token is stored in the public project files.

## Cilium eBPF vs Falco eBPF

Cilium uses eBPF for networking, service routing, policy, and network observability.

Falco uses eBPF for runtime syscall/event visibility such as process execution, file access, and socket activity.

They complement each other.

## Dashboard data model limitation

The native Falco rule counter is:

```text
falcosecurity_falco_rules_matches_total
```

Useful labels include:

```text
rule_name
priority
source
tag_custom
```

Metric target labels such as `namespace="falco"` and `pod="falco-..."` refer to the Falco sensor pod, not necessarily the workload that caused the event.

Application-level details are present in Falco alert output. A future Loki integration could provide workload-level Grafana panels.
