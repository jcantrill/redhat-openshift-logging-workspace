# Suppress unused kubernetes_logs annotation fields

From the [Vector improvements investigation](README.md)
([LOG-10547](https://redhat.atlassian.net/browse/LOG-10547)). Expected
benefit: per-event CPU and memory in the `kubernetes_logs` source and
the `viaq_0` remap.

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

## Plan

- Emit full explicit annotator specs with `""` for suppressed keys:
  `pod_ip`, `pod_ips`, `pod_owner`, `node_labels`, `pod_node_name`,
  `container_image_id`.
- Remove the now-dead `del`s in `viaq_0`.
- Remove the stale `.kubernetes.container_image_id` datamodel entry.

This is part of the approved CLO PR together with the
[label-dedotting fix](label-dedotting.md).
