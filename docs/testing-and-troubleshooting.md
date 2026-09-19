# Testing and Troubleshooting

## Controlled test philosophy

Tests were performed against an owned workload in the `dashboard` namespace. `kubectl exec` was used only to trigger runtime behavior. ArgoCD-managed deployment configuration was not modified.

## Interactive shell test

```bash
kubectl -n dashboard exec -it \
  backend-5b94fccfcc-mzp24 -- sh
```

Then:

```bash
exit
```

Expected alert:

```text
Interactive shell started in container
```

Observed context included `user=appuser`, `command=sh`, the backend pod, image, and `namespace=dashboard`.

## Service-account token read test

```bash
kubectl -n dashboard exec \
  backend-5b94fccfcc-mzp24 -- \
  sh -c 'cat /var/run/secrets/kubernetes.io/serviceaccount/token >/dev/null'
```

Expected alert:

```text
Sensitive credential file read in container
```

## Privilege-changing tool test

```bash
kubectl -n dashboard exec \
  backend-5b94fccfcc-mzp24 -- \
  su --version
```

Expected alert:

```text
Privilege-changing tool executed in container
```

## Verify Falco alerts

```bash
kubectl -n falco logs \
  -l app.kubernetes.io/name=falco \
  -c falco \
  --since=5m \
  | grep 'Interactive shell started in container'
```

Why each part is used:

- `-l ...`: selects all Falco pods by label
- `-c falco`: reads the Falco container
- `--since=5m`: limits the log window
- `grep`: filters for the alert text

## Verify rule counters in Prometheus

Start the port-forward:

```bash
kubectl -n monitoring port-forward \
  svc/kube-prometheus-stack-prometheus \
  9090:9090
```

Check readiness:

```bash
curl -sS http://127.0.0.1:9090/-/ready
```

Query all custom rule counters:

```bash
curl -sSG 'http://127.0.0.1:9090/api/v1/query' \
  --data-urlencode \
  'query=sum by (rule_name) (falcosecurity_falco_rules_matches_total{tag_custom="true"})'
```

After the final controlled tests, the three representative custom rules each had a count of `1`.

## Direct Falco metrics check

```bash
kubectl -n falco port-forward \
  svc/falco-metrics \
  8765:8765
```

Then:

```bash
curl -s http://127.0.0.1:8765/metrics | head -40
```

## Dashboard validation

Before applying:

```bash
kubectl apply --dry-run=client \
  -f grafana/falco-runtime-security-dashboard.yaml
```

Apply:

```bash
kubectl apply \
  -f grafana/falco-runtime-security-dashboard.yaml
```

The existing Grafana sidecar discovers ConfigMaps labeled `grafana_dashboard: "1"`.

## PromQL: total custom detections

```promql
sum(
  falcosecurity_falco_rules_matches_total{
    tag_custom="true"
  }
)
```

## PromQL: top custom rules

```promql
topk(
  10,
  sum by (rule_name) (
    falcosecurity_falco_rules_matches_total{
      tag_custom="true"
    }
  )
)
```

## PromQL: event rate

```promql
sum(rate(falcosecurity_scap_n_evts_total[5m]))
```

During validation this was roughly 15k processed events/sec across the cluster. This is event-processing throughput, not alert volume.

## PromQL: buffer drops

```promql
sum(increase(falcosecurity_scap_n_drops_buffer_total[$__range]))
```

Observed buffer drops were `0` during the validation window.

## Troubleshooting: fewer than 10 rules rendered

Symptom: `helm template` displayed fewer than 10 custom rules.

Diagnostic command:

```bash
grep -E '^    - rule: Custom ' \
  /tmp/falco-rendered.yaml
```

Cause: two rules were indented incorrectly inside the custom rule block.

Fix: align each `- rule:` entry at the same indentation level.

## Troubleshooting: CrashLoopBackOff after rule edit

Symptom: a Falco pod entered `CrashLoopBackOff` after an upgrade.

Diagnosis:

```bash
kubectl -n falco logs \
  <failing-pod> \
  -c falco \
  --previous
```

Falco reported:

```text
Value must be non-empty
```

Inspecting the rendered rule:

```bash
grep -nA40 -B5 \
  'Custom Suspicious System Modification or Temp Execution' \
  /tmp/falco-rendered.yaml
```

showed:

```yaml
condition: >
condition: >
```

Deleting the duplicate condition fixed the startup failure.

## Why `--previous` matters

When a container repeatedly restarts, the most useful startup failure may be in the previous container instance. `kubectl logs --previous` retrieves that failed instance's logs.

## Troubleshooting: noisy service-account rule

Initial behavior: legitimate token reads from ArgoCD, Cilium, and CoreDNS generated alerts.

Fix: restrict the service-account-token portion of the rule to shell/CLI-style processes.

Result: background noise disappeared and the deliberate `cat` test still triggered.

## Troubleshooting: noisy system modification rule

Initial behavior: `/bin/prometheus-config-reloader` generated repeated alerts while writing expected generated config files.

Fix: exclude only the specific process, `monitoring` namespace, and known generated paths.

Result: legitimate reload activity stopped alerting without excluding unrelated activity in the namespace.

## Useful verification commands

Falco pods:

```bash
kubectl -n falco get pods -o wide
```

Service and ServiceMonitor:

```bash
kubectl -n falco get svc,servicemonitor
```

List loaded custom rules:

```bash
kubectl -n falco exec <falco-pod> -c falco -- \
  falco -L | grep '^Custom '
```

Check Prometheus ServiceMonitor selector:

```bash
kubectl -n monitoring get prometheus -o yaml \
  | grep -A8 -B2 'serviceMonitorSelector'
```

Grafana service:

```bash
kubectl -n monitoring get svc | grep -i grafana
```

## Final validation state

- all 3 Falco pods healthy
- all 10 custom rules loaded
- Prometheus scraping all 3 Falco instances
- Grafana dashboard provisioned from ConfigMap
- 3 controlled detections visible
- Falco event rate visible
- 0 buffer drops during validation
- Prometheus config-reloader noise tuned out
