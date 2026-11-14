# SPIFFE registration in `Default` mode

| | |
| --- | --- |
| **What this is** | The SPIFFE gRPC registration as it works in the `Default` install mode, from setup to deletion. |
| **The whole design** | [README.md](README.md). This page repeats only what you need for `Default` and links there for the rest. |
| **Other modes** | [Singleton](mode-singleton.md), [Hosted](mode-hosted.md), [SingletonHosted](mode-singletonhosted.md) |

## 1. The mode in one paragraph

In `Default` mode everything runs on the managed cluster. The Klusterlet operator deploys
**two agents** there, the registration agent and the work agent, each in its own pod. Each agent
gets **its own SVID** from the SPIRE agent on its node and connects to the hub with it. This is
the most common layout and the one the design was written for first.

## 2. What runs where

| Cluster | What runs there for this cluster |
| --- | --- |
| Hub | gRPC server with the SPIFFE listener on port 8091; registration controllers with the delete guard. The same in every mode ([README 6.1](README.md#61-set-up-the-hub)). |
| Managed cluster | Klusterlet operator; registration agent and work agent in `open-cluster-management-agent`; SPIRE agent with the SPIFFE CSI driver; admission policies; add-on agents. |
| SPIRE server | Outside the clusters, run by the platform team. |

## 3. The agents and their identities

For a Klusterlet named `klusterlet` and cluster `cluster1`, with the default ID template:

| Agent | Deployment | ServiceAccount | `spiffe-role` label | SPIFFE ID | OCM user on the hub |
| --- | --- | --- | --- | --- | --- |
| Registration | `klusterlet-registration-agent` | `klusterlet-registration-sa` | `registration` | `spiffe://<td>/ocm/cluster/cluster1/agent/registration` | `system:open-cluster-management:cluster1:registration` |
| Work | `klusterlet-work-agent` | `klusterlet-work-sa` | `work` | `spiffe://<td>/ocm/cluster/cluster1/agent/work` | `system:open-cluster-management:cluster1:work` |

Both are in the groups `system:open-cluster-management:cluster1` and
`system:open-cluster-management:managed-clusters`. `<td>` is the trust domain of the SPIRE that
serves the managed cluster.

The two agents share nothing: no Secret, no file. With the `grpc` driver the registration agent
writes the hub credentials into a Secret for the work agent; here each one has its own SVID.

## 4. Set it up

### 4.1 The hub

The same in every mode. See [README 6.1](README.md#61-set-up-the-hub) or run:

```shell
clusteradm init --registration-drivers spiffe --spiffe-trust-domain ocm-hub.example.org
```

### 4.2 Join

On the managed cluster. No token:

```shell
clusteradm join --registration-auth spiffe --cluster-name cluster1 \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server --wait
```

Or create the `Klusterlet` yourself with `deployOption.mode: Default` and
`authType: spiffe` ([README 5.3](README.md#53-on-the-managed-cluster-klusterlet)).

The operator then, in this order:

1. installs the three admission policies on the managed cluster (section 5);
2. cleans `open-cluster-management-agent` and writes the record to `status.spiffe`;
3. sets `SPIFFEAdmissionReady=True` and writes both IDs to `status.spiffe.agentIDs`;
4. starts the two agents. They wait for their SVIDs.

### 4.3 Create the SPIRE entries

Only after `SPIFFEAdmissionReady=True`. Before that, anyone who can start a pod with the right
label and ServiceAccount would get an agent's SVID.

```shell
kubectl get klusterlet klusterlet -o jsonpath='{.status.spiffe.agentIDs}'
```

Two entries, one per agent. Each selects on namespace, role label **and** ServiceAccount:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster1-registration
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/registration"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: registration
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-registration-sa"
---
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster1-work
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: work
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-work-sa"
```

There is no `klusterlet` entry in this mode. More on entries: [`../clusterspiffeid.md`](../clusterspiffeid.md).

### 4.4 Accept

```shell
clusteradm accept --clusters cluster1
```

The registration agent created `cluster1` when it joined, and the hub pinned it to the trust
domain of its SVID. Until acceptance, the work agent is refused at the cluster gate
([README 6.3](README.md#63-accept-a-cluster)).

## 5. The admission policies

All three are on the managed cluster, the same cluster as the agents.

| Policy | Name | What it does here |
| --- | --- | --- |
| Agent namespace | `ocm-spiffe-agents-klusterlet` | In `open-cluster-management-agent`, only the operator changes the two Deployments. Pods with role `registration` must use `klusterlet-registration-sa`, and pods with role `work` must use `klusterlet-work-sa`. No `exec`, `attach`, debug container or image change. |
| Add-on namespaces | `ocm-spiffe-addons-klusterlet` | In namespaces labeled `open-cluster-management.io/spiffe-addon-namespace: klusterlet`, only the work agent creates workloads, and add-on pods use role `addon` and their own ServiceAccount. |
| Namespace label | `ocm-spiffe-addon-namespace-label` | Shared by all Klusterlets on the cluster. The add-on label can be set only when a namespace is created. |

If a policy is deleted or changed by someone else, the operator restores it and replaces every
agent and add-on pod ([README 6.11](README.md#611-recover-when-a-policy-was-changed)). The
operator watches the policies, so this starts within seconds.

## 6. The lifecycle

| Step | What happens in `Default` mode | More |
| --- | --- | --- |
| Send work | The work agent watches `cluster1`'s `ManifestWork`s with its `work` SVID, applies them on the managed cluster and sends the status back. | [README 6.4](README.md#64-send-work-to-a-cluster) |
| Renew | SPIRE renews each agent's SVID separately. When an old SVID expires, the hub closes that agent's connection and the agent reconnects with the new one. The other agent is not affected. | [README 6.5](README.md#65-renew-an-svid) |
| Deny (`hubAcceptsClient: false`) | The hub closes both connections. The gate then refuses the work agent, so no work reaches the cluster. The registration agent may still read its own `ManagedCluster`, to see when it is accepted again. | [README 6.6](README.md#66-deny-a-cluster-or-disable-an-add-on) |
| Delete | Remove the two SPIRE entries, then delete the `ManagedCluster`. The hub holds the name for `maxSVIDLifetime + maxClockSkew`. | [README 6.7](README.md#67-delete-a-cluster) |
| Add-ons | Add-on agents run on the managed cluster, in a namespace labeled for `klusterlet`, each with its own SVID. Their entries go on the managed cluster. | [README 6.8](README.md#68-run-an-add-on) |
| Remove the Klusterlet | The operator deletes its two policies and bindings, and the shared label policy if no other Klusterlet uses `spiffe`. | [README 6.14](README.md#614-move-an-existing-cluster-to-spiffe-and-back) |

**Why `Default` is the cleanest mode for denial.** Registration and work have separate SVIDs, so
the hub can cut off work while still letting the registration agent see when the cluster is
accepted again. In `Singleton` mode this split is not possible
([mode-singleton.md](mode-singleton.md#7-known-limits)).

## 7. Known limits

Each limit below says what it is in simple words, shows an example, explains why it happens and
what you can do about it.

### 7.1 Short SVIDs make updates arrive late

**In simple words.** If SPIRE gives the agents SVIDs that live less than about 10 minutes, the
agents reconnect to the hub very often. The client then starts to wait longer before each
reconnect, and while it waits, changes from the hub don't reach the cluster. Nothing is lost,
but an update can arrive up to about a minute late.

**Example.** The SPIRE entry for `cluster1` has `ttl: 2m`.

| Time | What happens |
| --- | --- |
| 10:00:00 | The agent connects with SVID #1, valid until 10:02. |
| 10:02:00 | SVID #1 expires. The hub closes the connection, as designed ([README 6.5](README.md#65-renew-an-svid)). |
| 10:02:07 | The client waits 5 to 10 seconds, then reconnects with SVID #2. |
| 10:04:00 | SVID #2 expires and the connection is closed again. This time the client waits 25 to 50 seconds, because it reconnected less than 10 minutes ago. |
| 10:04:10 | A hub user changes a `ManifestWork` for `cluster1`. |
| 10:04:40 | The agent reconnects, resyncs and gets the change: **30 seconds late**. |

In the e2e test (2-minute SVIDs, 13 updates over 5 minutes) one update came 42 seconds late. All
13 arrived.

**Why it happens.** The client that OCM's agents use to talk to the hub treats every closed
connection as a failure. After a failure it waits before trying again: 5 to 10 seconds the first
time, then 25 to 50 seconds, up to 60 seconds. It goes back to the short wait only after 10
minutes without a reconnect. With short SVIDs the connection is closed every few minutes, so the
wait never goes back to short.

**What you can do.** Give agent SVIDs a lifetime of 10 minutes or more. SPIRE's default X.509
lifetime is one hour, and with that this doesn't happen: each reconnect has only the normal 5 to
10 second gap.

**Status.** Open, `questions.md` Q18. A change to the client in `sdk-go` is proposed: don't count
a close at SVID expiry as a failure.

### 7.2 A custom SPIFFE ID layout is not tested on real clusters

**In simple words.** You can choose your own layout for the agents' SPIFFE IDs
([README 6.13](README.md#613-use-your-own-spiffe-id-layout)). That is tested in unit and
integration tests, but not yet end to end on real clusters with real SPIRE. And changing the
layout on a hub that is already running cuts off every cluster for a while.

**Example.** A company names its workloads
`spiffe://ocm-hub.example.org/k8s/<cluster>/ns/<namespace>/ocm/<role>`. They set this as
`agentIDTemplate` on the `ClusterManager` of a hub that already has 50 clusters.

1. The hub now accepts only IDs in the new layout. All 50 clusters' agents still have SVIDs in
   the old layout (`.../ocm/cluster/<cluster>/agent/<role>`).
2. Their connections are refused with `matches neither agentIDTemplate`.
3. Each cluster comes back only after its `Klusterlet` has the same template and its SPIRE
   entries give the new IDs.

**Why it happens.** The hub reads the cluster name and the role from the ID with **one**
template. It can't accept two layouts at the same time, because then one ID could be read in two
ways.

**What you can do.** Pick the layout before the first cluster joins. If you must change it, do
it in this order for the least downtime: add the new SPIRE entries first (agents then hold both
SVIDs), change the `ClusterManager`, change each `Klusterlet` quickly, and then delete the old
entries ([`../clusterspiffeid.md`](../clusterspiffeid.md#custom-id-templates)).

**Status.** The e2e check for this (Day 27, Task 7) was not run. `questions.md` Q7.

## 8. Test status

e2e on kind with SPIRE 1.15.5 ([`../e2e-results.md`](../e2e-results.md), cluster `cluster1`):

| Check | Result |
| --- | --- |
| Join with no token, entries after `SPIFFEAdmissionReady`, accept | pass |
| No hub credential Secret on the managed cluster | pass |
| Cluster pinned to the trust domain | pass |
| `ManifestWork` applied and `Available` | pass |
| Rotation under load (2-minute SVIDs, 5 minutes of updates) | nothing lost; one update 42 s late |
| Agent down longer than its SVID | covered by the rotation check |
| Cluster deny | pass (after fix ocm `2bdd76cff`) |
| Cluster delete: guard holds and releases on time | pass |
| `exec` into an agent refused | pass |
| Klusterlet delete removes its policies | pass (after fix ocm `8d1a4d175`) |
| Status feedback, add-on enable and revoke | not run |

Unit tests for this mode: the `Default` case in
[`klusterlet_spiffe_test.go`](../../ocm/pkg/operator/operators/klusterlet/controllers/klusterletcontroller/klusterlet_spiffe_test.go).
