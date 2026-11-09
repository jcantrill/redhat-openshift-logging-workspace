# Suppress unused kubernetes_logs annotation fields

From the [Vector improvements investigation](README.md)
([LOG-10547](https://redhat.atlassian.net/browse/LOG-10547)). Expected
benefit: per-event CPU and memory in the `kubernetes_logs` source and
the `viaq_0` remap.

**Status: validated on a live cluster (2026-11-09) — see Results.**

## Problem

The CLO spells only 5 of the ~15 `FieldsSpec` keys in the
`kubernetes_logs` source config; the rest fall back to Vector defaults.
`pod_ip`, `pod_ips`, `pod_owner`, `node_labels`, `pod_node_name`, and
`container_image_id` are inserted per event and then deleted in `viaq_0`
(or, for `pod_node_name`, overwritten by the `VECTOR_SELF_NODE_NAME` env
var before anything reads it). Every FieldsSpec key supports `""` to
suppress insertion entirely.

Also identified here (correctness):

- Redundant hostname: `kubernetes_logs` `pod_node_name → hostname` and
  the audit file sources' `host_key = "hostname"` are both dead-written —
  the `_meta` remap overwrites `_internal.hostname` from the
  `VECTOR_SELF_NODE_NAME` env var before anything reads it.
- `.kubernetes.container_image_id` is documented in the VIAQ datamodel
  (`docs/reference/datamodels/viaq/v1.adoc`) but deleted in `viaq_0`.
  Decision: suppress it and remove the stale datamodel entry.

## Empirical inventory (live cluster, RHOL 6.7, default config)

Per-event payload of an application record delivered to Loki:
**2222 bytes total**; `kubernetes.annotations` alone is 878 bytes (40%)
— mostly the OVN `pod-networks` and `network-status` annotation JSON.
`kubernetes.namespace_labels` 248 B, `labels` 148 B.

Confirmed field behavior against the datamodel and the normalize
filter:

| Field | Behavior on 6.7 | Datamodel |
|---|---|---|
| `pod_ip` (13 B) | **ships to Loki** | not documented — leak |
| `pod_owner` (35 B) | **ships to Loki** | not documented — leak |
| `pod_ips` | attached, deleted in normalize | not documented |
| `container_image_id` | attached, deleted in normalize | documented (stale) |
| `node_labels` | attached, deleted in normalize | not documented |
| `pod_node_name` → `hostname` | overwritten by `VECTOR_SELF_NODE_NAME` | (as `.kubernetes.host`) |

## Results (A/B, 4k events/s aggregate, 4 generator pods)

Method: RHOL 6.7 from Red Hat Operators, LokiStack 1x.demo with garage
S3 storage (CLO `hack/manifests`), CLF application→lokiStack. Side A =
default config; Side B = same config with the six fields suppressed in
`kubernetes_logs` (config patched live via CLF `managementState:
Unmanaged` + configmap edit + pod restart; the same change is on CLO
bench worktree branch `bench-rotation-fix-6.7`, commit `03fd487e5`).
5-minute steady window per side, `oc adm top` every 30s. Rate capped at
4k/s because Loki 1x.demo cannot sustain 20k/s against garage/gp3
storage (ingester 500s — same distress mode as the original LOG-10388
bench; sink backpressure stalls the pipeline and poisons CPU numbers).

| Metric (3 hot collector pods) | Default | Suppressed | Δ |
|---|---|---|---|
| CPU median (per pod) | 155m | 130m | **−16%** |
| CPU sum (hot pods) | 598m | 497m | −17% |
| Memory median (per pod) | 400Mi | 280Mi | **−30%** |
| Output payload per event | 2222 B | 2175 B | −47 B (leaks closed) |

Verification: `pod_ip`/`pod_owner` absent from Loki records; `hostname`
still populated (via `VECTOR_SELF_NODE_NAME`) — the source's
`pod_node_name` write was confirmed dead.

Consistent with the earlier node-labels-only experiment at 20k/s
(−19% CPU / −55% mem, measured during the original bench with Loki
distress) — the effect is real and roughly additive across the field
set. Caveats: single 5-minute window per side, rate limited to 4k/s by
the Loki backend, pod placement differs slightly between sides.

## Plan

- Emit full explicit annotator specs with `""` for suppressed keys:
  `pod_ip`, `pod_ips`, `pod_owner`, `node_labels`, `pod_node_name`,
  `container_image_id`.
- Remove the now-dead `del`s in `viaq_0`.
- Remove the stale `.kubernetes.container_image_id` datamodel entry.

This is part of the approved CLO PR together with the
[label-dedotting fix](label-dedotting.md).
