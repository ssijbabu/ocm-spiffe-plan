# Day 29: real cluster check on AKS

**Goal:** run the Default-mode lifecycle on the AKS clusters, including an agent that comes back
after its SVID expired, and confirm the managed-platform assumptions the design relies on.

**Clusters:** `ocm-hub` and `ocm-spoke`, resource group `ocm-spiffe-rg`. They are stopped.

## Before you start

| Input | Where |
| --- | --- |
| SPIRE on ocm-hub | `spire-crds` 0.6.1, `spire` 0.30.4, namespace `spire-server`, trust domain `ocm-hub.example.org`, `k8s_psat` (see `CLAUDE.md`) |
| Images | push the `spire` builds to a registry the clusters can pull from |
| Cost | the clusters cost money while running; stop them at the end of the day |

## Task 1: start the clusters

```shell
az aks start -g ocm-spiffe-rg -n ocm-hub
az aks start -g ocm-spiffe-rg -n ocm-spoke
az aks get-credentials -g ocm-spiffe-rg -n ocm-hub --context ocm-hub --overwrite-existing
az aks get-credentials -g ocm-spiffe-rg -n ocm-spoke --context ocm-spoke --overwrite-existing
kubectl --context ocm-hub -n spire-server get pods
```

**Output:** SPIRE server, agents, CSI driver and controller manager all `Running`.

## Task 2: confirm the controller identities

The admission policy needs each built-in controller to use its own ServiceAccount
(`--use-service-account-credentials=true`). AKS doesn't expose kube-controller-manager flags, so
check the behaviour instead.

```shell
kubectl --context ocm-spoke create ns sa-check
kubectl --context ocm-spoke -n sa-check create deploy probe --image=registry.k8s.io/pause:3.10
kubectl --context ocm-spoke get --raw "/apis/apps/v1/namespaces/sa-check/replicasets" \
  | jq -r '.items[0].metadata.managedFields[] | "\(.manager) \(.operation)"'
```

`managedFields` shows the field manager, not the user. To see the user, use an audit-only policy:
install a ValidatingAdmissionPolicy in `sa-check` with `validationActions: [Audit]` whose message
includes `request.userInfo.username`, or temporarily a `Deny` that always fails, and read the
username from the error event of the ReplicaSet controller.

**Expected:** ReplicaSets are created by `system:serviceaccount:kube-system:deployment-controller`
and Pods by `system:serviceaccount:kube-system:replicaset-controller`.

If they aren't, write it down: the design's statement "managed platforms must confirm it" applies,
and AKS isn't supported for `spiffe` mode until it does. Delete `sa-check` afterwards.

## Task 3: Default-mode lifecycle

Repeat Day 27, Tasks 1 to 3, against `ocm-hub` and `ocm-spoke`. The hub's gRPC endpoint needs an
internal load balancer reachable from the spoke (all SPIRE Services are ClusterIP today; see
`CLAUDE.md`).

## Task 4: agent down past SVID expiry

This answers the earlier question "the agent was down, its certificate expired, and it came back:
does it get a new one?"

| Step | Input | Expected |
| --- | --- | --- |
| 1 | set the work entry's TTL to 10m | new SVIDs live 10m |
| 2 | `kubectl scale deploy/klusterlet-work-agent --replicas=0` for 15m | the old SVID has expired |
| 3 | scale back to 1 | the new pod gets a fresh SVID from the SPIRE agent at start |
| 4 | apply a ManifestWork | applied within a minute, no manual step |

Record the work agent's log lines showing the new SVID's serial and expiry.

## Task 5: record and stop

Write the results to `plan/aks-results.md` (pass/fail per step, and any difference from kind).

```shell
az aks stop -g ocm-spiffe-rg -n ocm-spoke
az aks stop -g ocm-spiffe-rg -n ocm-hub
az aks show -g ocm-spiffe-rg -n ocm-hub --query powerState.code -o tsv
```

**Output:** `Stopped` for both.
