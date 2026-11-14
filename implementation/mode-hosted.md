# SPIFFE registration in `Hosted` mode

| | |
| --- | --- |
| **What this is** | The SPIFFE gRPC registration as it works in the `Hosted` install mode, from setup to deletion. |
| **The whole design** | [README.md](README.md). This page repeats only what you need for `Hosted` and links there for the rest. |
| **Other modes** | [Default](mode-default.md), [Singleton](mode-singleton.md), [SingletonHosted](mode-singletonhosted.md) |

## 1. The mode in one paragraph

In `Hosted` mode the agents don't run on the cluster they manage. They run on a separate
**management cluster**, in a namespace named after the Klusterlet, and reach the managed cluster
through a kubeconfig. One management cluster can host the agents of many managed clusters. As in
`Default` mode there are **two agents**, registration and work, each with **its own SVID**. The
SVIDs come from the SPIRE agent on the **management** cluster, because that is where the agent
pods run.

## 2. What runs where

| Cluster | What runs there for this cluster |
| --- | --- |
| Hub | gRPC server with the SPIFFE listener on port 8091; registration controllers with the delete guard. The same in every mode ([README 6.1](README.md#61-set-up-the-hub)). |
| Management cluster | Klusterlet operator and the `Klusterlet`; registration agent and work agent in the namespace `<klusterlet>`; SPIRE agent with the SPIFFE CSI driver; the agent namespace policy; the `external-managed-kubeconfig` Secret. |
| Managed cluster | What the work agent applies; add-on agents; the add-on namespace policy and the namespace label policy. A SPIRE agent only if add-ons run there (section 6). |
| SPIRE server | Outside the clusters, run by the platform team. |

## 3. The agents and their identities

For a Klusterlet named `cluster4` and cluster `cluster4`, with the default ID template:

| Agent | Deployment (on the management cluster) | ServiceAccount | `spiffe-role` label | SPIFFE ID | OCM user on the hub |
| --- | --- | --- | --- | --- | --- |
| Registration | `cluster4-registration-agent` | `cluster4-registration-sa` | `registration` | `spiffe://<mgmt td>/ocm/cluster/cluster4/agent/registration` | `system:open-cluster-management:cluster4:registration` |
| Work | `cluster4-work-agent` | `cluster4-work-sa` | `work` | `spiffe://<mgmt td>/ocm/cluster/cluster4/agent/work` | `system:open-cluster-management:cluster4:work` |

`<mgmt td>` is the trust domain of the SPIRE that serves the **management** cluster. The
`{cluster}` part of the ID is still the managed cluster's name; that is what the hub uses.

**The namespace keeps clusters apart.** Each hosted Klusterlet has its own agent namespace,
named after the Klusterlet. SPIRE entries select on that namespace, so `cluster5`'s pods on the
same management cluster can never get `cluster4`'s IDs. Don't use `{{ .PodMeta.Namespace }}` for
the cluster name in an entry.

**Names with clusteradm.** `clusteradm join --mode hosted` names the Klusterlet
`klusterlet-hosted-<6 random characters>`, and the agent namespace has the same name. Read the
real names from the Klusterlet before writing the entries (section 4.3).

## 4. Set it up

### 4.1 The hub

The same in every mode. See [README 6.1](README.md#61-set-up-the-hub) or run:

```shell
clusteradm init --registration-drivers spiffe --spiffe-trust-domain ocm-hub.example.org
```

If the management cluster has its own SPIRE in another trust domain, add it with
`--spiffe-agent-trust-domains` and federate the two SPIREs
([README 6.12](README.md#612-clusters-with-their-own-spire)).

### 4.2 Join

On the **management** cluster. No hub token, but the managed cluster's kubeconfig:

```shell
clusteradm join --registration-auth spiffe --cluster-name cluster4 \
  --mode hosted --managed-cluster-kubeconfig ./cluster4.kubeconfig \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server --wait
```

Or create the `Klusterlet` on the management cluster with `deployOption.mode: Hosted` and
`authType: spiffe` ([README 5.3](README.md#53-on-the-managed-cluster-klusterlet)), plus the
`external-managed-kubeconfig` Secret in the agent namespace.

The operator then, in this order:

1. installs the agent namespace policy on the **management** cluster, and the add-on and label
   policies on the **managed** cluster (section 5);
2. cleans the agent namespace and writes the record to `status.spiffe`;
3. sets `SPIFFEAdmissionReady=True` and writes both IDs to `status.spiffe.agentIDs`;
4. starts the two agents on the management cluster. They wait for their SVIDs.

### 4.3 Create the SPIRE entries

On the management cluster, only after `SPIFFEAdmissionReady=True`.

```shell
kubectl get klusterlet                                  # the Klusterlet's name = the agent namespace
kubectl get klusterlet <name> -o jsonpath='{.status.spiffe.agentIDs}'
```

Two entries per hosted cluster. Here the management cluster is in `east.example.org`, so the
entries federate with the hub's trust domain:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster4-registration
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster4/agent/registration"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: cluster4
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: registration
  workloadSelectorTemplates:
  - "k8s:sa:cluster4-registration-sa"
---
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster4-work
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster4/agent/work"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: cluster4
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: work
  workloadSelectorTemplates:
  - "k8s:sa:cluster4-work-sa"
```

If the management cluster is in the hub's trust domain, use that trust domain and leave out
`federatesWith`. More on entries: [`../clusterspiffeid.md`](../clusterspiffeid.md).

### 4.4 Accept

```shell
clusteradm accept --clusters cluster4
```

The registration agent created `cluster4` when it joined, and the hub pinned it to the trust
domain of its SVID, that is, the management cluster's trust domain
([README 6.3](README.md#63-accept-a-cluster)).

## 5. The admission policies

The policies are split across the two clusters, because the agents and the add-ons run in
different places.

| Policy | Name | On | What it does here |
| --- | --- | --- | --- |
| Agent namespace | `ocm-spiffe-agents-cluster4` | management cluster | In the namespace `cluster4`, only the operator changes the two Deployments. Role `registration` must use `cluster4-registration-sa`, role `work` must use `cluster4-work-sa`. No `exec`, `attach`, debug container or image change. |
| Add-on namespaces | `ocm-spiffe-addons-cluster4` | managed cluster | In namespaces labeled `open-cluster-management.io/spiffe-addon-namespace: cluster4`, only `cluster4`'s work agent creates workloads. |
| Namespace label | `ocm-spiffe-addon-namespace-label` | managed cluster | The add-on label can be set only when a namespace is created. |

**One Klusterlet per label value.** Many Klusterlets share a management cluster. Each one's
policies, records and recovery cover only its own namespace and its own add-on namespaces. When
one Klusterlet's policy is changed, recovery replaces only that Klusterlet's pods.

**Recovery** is the same as in every mode
([README 6.11](README.md#611-recover-when-a-policy-was-changed)), with one difference: the
operator watches policies only on the cluster it runs on. A change to the agent policy on the
management cluster is seen within seconds. A change to the two policies on the managed cluster is
seen at the next resync, within five minutes (`questions.md` Q20).

## 6. The lifecycle

| Step | What happens in `Hosted` mode | More |
| --- | --- | --- |
| Send work | The work agent on the management cluster watches `cluster4`'s `ManifestWork`s with its `work` SVID and applies them on the **managed** cluster, through its kubeconfig. Nothing lands on the management cluster. | [README 6.4](README.md#64-send-work-to-a-cluster) |
| Renew | SPIRE on the management cluster renews each agent's SVID. The hub closes and the agent reopens the connection, as in every mode. | [README 6.5](README.md#65-renew-an-svid) |
| Deny | As in `Default`: both connections closed, the work agent refused, the registration agent may still read its `ManagedCluster`. | [README 6.6](README.md#66-deny-a-cluster-or-disable-an-add-on) |
| Delete | Remove the two entries on the management cluster, then delete the `ManagedCluster`. The hub holds the name for `maxSVIDLifetime + maxClockSkew`. | [README 6.7](README.md#67-delete-a-cluster) |
| Add-ons | An add-on deployed the usual way runs on the **managed** cluster, in a namespace labeled `cluster4`. Its SVID comes from a SPIRE agent there, so the managed cluster needs a SPIRE agent, the CSI driver and the add-on's entry. | [README 6.8](README.md#68-run-an-add-on) |
| Remove the Klusterlet | The operator deletes its policies and bindings on both clusters, and the shared label policy if no other Klusterlet uses `spiffe`. | [README 6.14](README.md#614-move-an-existing-cluster-to-spiffe-and-back) |

## 7. Known limits

| Limit | Details |
| --- | --- |
| **The managed cluster's kubeconfig is a Secret** | `external-managed-kubeconfig` on the management cluster holds the kubeconfig the agents use to reach the managed cluster. It is not a hub credential and every driver needs it in this mode, but it is long-lived. Whether that is in scope of "no long-lived secrets" is open (`questions.md` Q15). |
| Policies on the managed cluster | Checked only at the five-minute resync (`questions.md` Q20). |
| Short SVIDs | With SVIDs that live less than 10 minutes, an update can arrive up to about a minute late after a reconnect (`questions.md` Q18). |

## 8. Test status

e2e on kind with SPIRE 1.15.5 ([`../e2e-results.md`](../e2e-results.md)): cluster `cluster4`, agents
on `ocm-e2e-spoke` (management), workloads on `ocm-e2e-managed`. The managed cluster had no SPIRE,
because no agent runs there.

| Check | Result |
| --- | --- |
| Join with no token, entries after `SPIFFEAdmissionReady`, accept | pass |
| No hub credential Secret (`external-managed-kubeconfig` exists, see Q15) | pass |
| Cluster pinned to the trust domain | pass |
| `ManifestWork` applied on the managed cluster, not the management cluster | pass |
| `exec` into an agent refused | pass |
| Tampered agent policy: recovery for this Klusterlet only | pass in 6 s (after fix ocm `301d209f2`); `cluster5` on the same management cluster untouched |
| Another Klusterlet's work agent in this one's add-on namespace | not run on real clusters |
| Label put on `default` during recovery | not run on real clusters; unit test only |
| Add-ons | not run (the managed clusters had no SPIRE) |
| Rotation, deny, delete | run in `Default` mode only; the hub side is the same |

Unit tests for this mode: the `Hosted` case in
[`klusterlet_spiffe_test.go`](../../ocm/pkg/operator/operators/klusterlet/controllers/klusterletcontroller/klusterlet_spiffe_test.go),
which also uses a namespace template and an agent trust domain of `east`.
