# Vector on OpenShift

Two-tier log and metrics pipeline:

    nodes: vector-agent (DaemonSet) ──vector/gRPC :6000──▶ vector-aggregator (StatefulSet x3) ──▶ Loki
                                                                              └─▶ Prometheus metrics :9598

## Files
| File | Purpose |
|---|---|
| namespace.yaml | `vector` namespace |
| serviceaccounts.yaml | SAs for agent and aggregator |
| scc.yaml | Custom SCC (hostPath, root, spc_t; not privileged, caps dropped) |
| rbac.yaml | Pod/namespace/node read + `use` on the SCC |
| agent-configmap.yaml / agent-daemonset.yaml | Node collector |
| aggregator-configmap.yaml / aggregator-statefulset.yaml | Central tier with disk buffer |
| services.yaml | Aggregator, headless, and agent-metrics services |
| pdb.yaml, networkpolicy.yaml | Availability and isolation |
| servicemonitor.yaml, prometheusrule.yaml | Monitoring and alerts |
| secret.example.yaml | Sink credentials template (apply separately) |

## Deploy
    oc apply -f secret.example.yaml        # after editing, or create via your secret manager
    oc apply -k .
    oc -n vector rollout status ds/vector-agent
    oc -n vector rollout status sts/vector-aggregator

## Before you apply, change
1. `CLUSTER_NAME` in agent-daemonset.yaml
2. Image: mirror to your internal registry and pin by digest (`oc adm` / ImageContentSourcePolicy or IDMS)
3. `storageClassName` in the aggregator volumeClaimTemplates
4. Sink in aggregator-configmap.yaml (Loki shown; swap for Elasticsearch, Kafka, Splunk HEC, S3...)
5. Enable user-workload monitoring if you want the ServiceMonitor/alerts to work

## Operational notes
- Agent runs as root with `spc_t` because RHCOS pod logs are root-owned with SELinux labels. This is why the SCC exists; do not use `privileged`.
- Agent buffer lives on the node at `/var/lib/vector`; delete it if you ever need to reset checkpoints.
- Do not run alongside the OpenShift Logging collector against the same sink without deduplication, or you will double-ingest.
- Config changes: `oc -n vector rollout restart ds/vector-agent sts/vector-aggregator`.
- Validate configs offline: `vector validate --no-environment vector.yaml`.
- Audit and journal logs are not collected here. Add a `journald` source (needs `/run/log/journal` mount) or use the OpenShift Logging stack for audit requirements.
