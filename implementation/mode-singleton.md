# SPIFFE registration in `Singleton` mode

| | |
| --- | --- |
| **What this is** | The SPIFFE gRPC registration as it works in the `Singleton` install mode, from setup to deletion. |
| **The whole design** | [README.md](README.md). This page repeats only what you need for `Singleton` and links there for the rest. |
| **Other modes** | [Default](mode-default.md), [Hosted](mode-hosted.md), [SingletonHosted](mode-singletonhosted.md) |

## 1. The mode in one paragraph

In `Singleton` mode everything runs on the managed cluster, like `Default`, but the Klusterlet
operator deploys **one agent** that does both registration and work. That agent has **one SVID**,
with the role `klusterlet`, and uses it for every request to the hub. It saves a pod and a
connection. The price is that the hub can't tell the agent's registration requests from its work
requests by identity (section 7).

## 2. What runs where

| Cluster | What runs there for this cluster |
| --- | --- |
| Hub | gRPC server with the SPIFFE listener on port 8091; registration controllers with the delete guard. The same in every mode ([README 6.1](README.md#61-set-up-the-hub)). |
| Managed cluster | Klusterlet operator; the one agent in `open-cluster-management-agent`; SPIRE agent with the SPIFFE CSI driver; admission policies; add-on agents. |
| SPIRE server | Outside the clusters, run by the platform team. |

## 3. The agent and its identity

For a Klusterlet named `klusterlet` and cluster `cluster1`, with the default ID template:

| Agent | Deployment | ServiceAccount | `spiffe-role` label | SPIFFE ID | OCM user on the hub |
| --- | --- | --- | --- | --- | --- |
| Registration and work | `klusterlet-agent` | `klusterlet-work-sa` | `klusterlet` | `spiffe://<td>/ocm/cluster/cluster1/agent/klusterlet` | `system:open-cluster-management:cluster1:klusterlet` |

Groups: `system:open-cluster-management:cluster1` and
`system:open-cluster-management:managed-clusters`, the same as in `Default` mode, so the hub's
permissions for the cluster are the same.

The agent reuses the work agent's ServiceAccount, `klusterlet-work-sa`. The role label
`klusterlet` is what tells it apart from a work agent, and the admission policy makes sure the
label and the ServiceAccount always come together.

## 4. Set it up

### 4.1 The hub

The same in every mode. See [README 6.1](README.md#61-set-up-the-hub) or run:

```shell
clusteradm init --registration-drivers spiffe --spiffe-trust-domain ocm-hub.example.org
```

### 4.2 Join

On the managed cluster. No token. Add `--singleton`:

```shell
clusteradm join --registration-auth spiffe --cluster-name cluster1 --singleton \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server --wait
```

Or create the `Klusterlet` yourself with `deployOption.mode: Singleton` and
`authType: spiffe` ([README 5.3](README.md#53-on-the-managed-cluster-klusterlet)).

The operator then, in this order:

1. installs the three admission policies on the managed cluster (section 5);
2. cleans `open-cluster-management-agent` and writes the record to `status.spiffe`;
3. sets `SPIFFEAdmissionReady=True` and writes the one ID to `status.spiffe.agentIDs`;
4. starts the agent. It waits for its SVID.

### 4.3 Create the SPIRE entry

Only after `SPIFFEAdmissionReady=True`.

```shell
kubectl get klusterlet klusterlet -o jsonpath='{.status.spiffe.agentIDs}'
```

One entry:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster1-klusterlet
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/klusterlet"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: klusterlet
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-work-sa"
```

There are no `registration` or `work` entries in this mode. If you switch a cluster from
`Default` to `Singleton`, delete the two old entries. More on entries:
[`../clusterspiffeid.md`](../clusterspiffeid.md).

### 4.4 Accept

```shell
clusteradm accept --clusters cluster1
```

The agent created `cluster1` when it joined, and the hub pinned it to the trust domain of its
SVID ([README 6.3](README.md#63-accept-a-cluster)).

## 5. The admission policies

All three are on the managed cluster, the same cluster as the agent.

| Policy | Name | What it does here |
| --- | --- | --- |
| Agent namespace | `ocm-spiffe-agents-klusterlet` | In `open-cluster-management-agent`, only the operator changes `klusterlet-agent`. A pod with role `klusterlet` must use `klusterlet-work-sa`. No `exec`, `attach`, debug container or image change. |
| Add-on namespaces | `ocm-spiffe-addons-klusterlet` | In namespaces labeled `open-cluster-management.io/spiffe-addon-namespace: klusterlet`, only the agent (as the work agent) creates workloads, and add-on pods use role `addon` and their own ServiceAccount. |
| Namespace label | `ocm-spiffe-addon-namespace-label` | Shared by all Klusterlets on the cluster. The add-on label can be set only when a namespace is created. |

Recovery after a policy change is the same as in every mode
([README 6.11](README.md#611-recover-when-a-policy-was-changed)).

## 6. The lifecycle

| Step | What happens in `Singleton` mode | More |
| --- | --- | --- |
| Send work | The agent watches `cluster1`'s `ManifestWork`s with its `klusterlet` SVID, applies them and sends the status back. The hub sees user `...:cluster1:klusterlet` for both registration and work requests. | [README 6.4](README.md#64-send-work-to-a-cluster) |
| Renew | SPIRE renews the one SVID. When the old one expires, the hub closes the agent's connections and it reconnects with the new SVID. Registration and work reconnect together. | [README 6.5](README.md#65-renew-an-svid) |
| Deny (`hubAcceptsClient: false`) | The hub closes the agent's connections. The gate does **not** refuse the agent afterwards, because it is also the registration agent and must see when the cluster is accepted again. See section 7. | [README 6.6](README.md#66-deny-a-cluster-or-disable-an-add-on) |
| Delete | Remove the one SPIRE entry, then delete the `ManagedCluster`. The hub holds the name for `maxSVIDLifetime + maxClockSkew`. | [README 6.7](README.md#67-delete-a-cluster) |
| Add-ons | Add-on agents run on the managed cluster, in a namespace labeled for `klusterlet`, each with its own SVID. Nothing about add-ons changes in this mode. | [README 6.8](README.md#68-run-an-add-on) |
| Remove the Klusterlet | The operator deletes its two policies and bindings, and the shared label policy if no other Klusterlet uses `spiffe`. | [README 6.14](README.md#614-move-an-existing-cluster-to-spiffe-and-back) |

## 7. Known limits

Each limit below says what it is in simple words, shows an example, explains why it happens and
what you can do about it.

### 7.1 A denied cluster can still get work

**In simple words.** When the hub admin denies a cluster (`hubAcceptsClient: false`), the hub
should stop sending it work. In this mode it doesn't: the one agent keeps receiving
`ManifestWork`s. This is the same as the `csr` driver in singleton mode today.

**Example.**

| Time | What happens |
| --- | --- |
| 10:00:00 | `cluster2` is accepted and running. |
| 10:01:00 | The hub admin sets `hubAcceptsClient: false` on `cluster2`. |
| 10:01:00 | The hub closes all of `cluster2`'s connections. |
| 10:01:07 | The agent reconnects with the same SVID (role `klusterlet`). The cluster gate lets it in. |
| 10:05:00 | A hub user creates a `ManifestWork` in namespace `cluster2`. |
| 10:05:00 | The agent receives it and applies it. **The deny didn't stop the work.** |

In `Default` mode the same steps end differently: at 10:01:07 the work agent (role `work`) is
refused with `cluster cluster2 is not accepted by the hub`, and the `ManifestWork` never arrives.

**Why it happens.** The hub decides by the role in the SVID.

- A denied cluster's **registration** agent must still be let in, so it can see when the cluster
  is accepted again.
- A denied cluster's **work** agent must be kept out.
- In this mode one agent does both, with one SVID and the role `klusterlet`. The gate can't keep
  out half of an agent, so it lets it in.
- Once it is in, the normal permission check (RBAC) decides. The hub keeps the cluster's work
  RoleBinding when a cluster is denied, so the agent's work requests still pass.

**What you can do.**

- If denial must stop work, use `Default` mode.
- To stop work for a denied cluster in this mode, also delete its `ManifestWork`s, or remove the
  cluster from its placements.
- To cut the cluster off completely, delete its SPIRE entry. The agent then can't get a new SVID,
  and is refused when its current one expires (at most the SVID lifetime later).

**Status.** Open, `questions.md` Q19. A fix would make the gate look at what each request is
about: allow only the cluster's own `ManagedCluster` and lease for a denied cluster, and refuse
`ManifestWork`s.

### 7.2 Short SVIDs make updates arrive late

**In simple words.** If SPIRE gives the agents SVIDs that live less than about 10 minutes, the
agents reconnect to the hub very often. The client then starts to wait longer before each
reconnect, and while it waits, changes from the hub don't reach the cluster. Nothing is lost,
but an update can arrive up to about a minute late.

**Example.** The SPIRE entry for `cluster2` has `ttl: 2m`.

| Time | What happens |
| --- | --- |
| 10:00:00 | The agent connects with SVID #1, valid until 10:02. |
| 10:02:00 | SVID #1 expires. The hub closes the connection, as designed ([README 6.5](README.md#65-renew-an-svid)). |
| 10:02:07 | The client waits 5 to 10 seconds, then reconnects with SVID #2. |
| 10:04:00 | SVID #2 expires and the connection is closed again. This time the client waits 25 to 50 seconds, because it reconnected less than 10 minutes ago. |
| 10:04:10 | A hub user changes a `ManifestWork` for `cluster2`. |
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

## 8. Test status

e2e on kind with SPIRE 1.15.5 ([`../e2e-results.md`](../e2e-results.md), cluster `cluster2`):

| Check | Result |
| --- | --- |
| Join with no token, entry after `SPIFFEAdmissionReady`, accept | pass |
| No hub credential Secret on the managed cluster | pass |
| Cluster pinned to the trust domain | pass |
| `ManifestWork` applied and `Available` | pass |
| `exec` into the agent refused | pass |
| Klusterlet delete removes its policies | pass |
| Rotation, deny, delete | run in `Default` mode only; the hub side is the same |
| Status feedback, add-on enable and revoke | not run |

Unit tests for this mode: the `Singleton` case in
[`klusterlet_spiffe_test.go`](../../ocm/pkg/operator/operators/klusterlet/controllers/klusterletcontroller/klusterlet_spiffe_test.go).
