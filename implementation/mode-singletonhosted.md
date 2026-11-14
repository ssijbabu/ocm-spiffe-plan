# SPIFFE registration in `SingletonHosted` mode

| | |
| --- | --- |
| **What this is** | The SPIFFE gRPC registration as it works in the `SingletonHosted` install mode, from setup to deletion. |
| **The whole design** | [README.md](README.md). This page repeats only what you need for `SingletonHosted` and links there for the rest. |
| **Other modes** | [Default](mode-default.md), [Singleton](mode-singleton.md), [Hosted](mode-hosted.md) |

## 1. The mode in one paragraph

`SingletonHosted` combines the other two variations. Like `Hosted`, the agent runs on a separate
**management cluster**, in a namespace named after the Klusterlet, and reaches the managed
cluster through a kubeconfig. Like `Singleton`, there is **one agent** that does registration and
work, with **one SVID** and the role `klusterlet`. The SVID comes from the SPIRE agent on the
management cluster.

## 2. What runs where

| Cluster | What runs there for this cluster |
| --- | --- |
| Hub | gRPC server with the SPIFFE listener on port 8091; registration controllers with the delete guard. The same in every mode ([README 6.1](README.md#61-set-up-the-hub)). |
| Management cluster | Klusterlet operator and the `Klusterlet`; the one agent in the namespace `<klusterlet>`; SPIRE agent with the SPIFFE CSI driver; the agent namespace policy; the `external-managed-kubeconfig` Secret. |
| Managed cluster | What the agent applies; add-on agents; the add-on namespace policy and the namespace label policy. A SPIRE agent only if add-ons run there (section 6). |
| SPIRE server | Outside the clusters, run by the platform team. |

## 3. The agent and its identity

For a Klusterlet named `cluster5` and cluster `cluster5`, with the default ID template:

| Agent | Deployment (on the management cluster) | ServiceAccount | `spiffe-role` label | SPIFFE ID | OCM user on the hub |
| --- | --- | --- | --- | --- | --- |
| Registration and work | `cluster5-agent` | `cluster5-work-sa` | `klusterlet` | `spiffe://<mgmt td>/ocm/cluster/cluster5/agent/klusterlet` | `system:open-cluster-management:cluster5:klusterlet` |

`<mgmt td>` is the trust domain of the SPIRE that serves the **management** cluster.

Two things from the other modes apply together:

- **From `Hosted`:** the agent namespace is named after the Klusterlet, and the SPIRE entry
  selects on it. That keeps this cluster's identity away from the other Klusterlets on the same
  management cluster. `clusteradm join --mode hosted` names the Klusterlet
  `klusterlet-hosted-<6 random characters>`; read the real name before writing the entry.
- **From `Singleton`:** the agent reuses the work ServiceAccount, and the role label `klusterlet`
  tells it apart from a work agent. The admission policy keeps the label and the ServiceAccount
  together.

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

On the **management** cluster, with both `--mode hosted` and `--singleton`:

```shell
clusteradm join --registration-auth spiffe --cluster-name cluster5 \
  --mode hosted --singleton --managed-cluster-kubeconfig ./cluster5.kubeconfig \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server --wait
```

Or create the `Klusterlet` on the management cluster with `deployOption.mode: SingletonHosted`
and `authType: spiffe` ([README 5.3](README.md#53-on-the-managed-cluster-klusterlet)), plus the
`external-managed-kubeconfig` Secret in the agent namespace.

The operator then, in this order:

1. installs the agent namespace policy on the **management** cluster, and the add-on and label
   policies on the **managed** cluster (section 5);
2. cleans the agent namespace and writes the record to `status.spiffe`;
3. sets `SPIFFEAdmissionReady=True` and writes the one ID to `status.spiffe.agentIDs`;
4. starts the agent on the management cluster. It waits for its SVID.

### 4.3 Create the SPIRE entry

On the management cluster, only after `SPIFFEAdmissionReady=True`.

```shell
kubectl get klusterlet                                  # the Klusterlet's name = the agent namespace
kubectl get klusterlet <name> -o jsonpath='{.status.spiffe.agentIDs}'
```

One entry. Here the management cluster is in `east.example.org`:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster5-klusterlet
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster5/agent/klusterlet"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: cluster5
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: klusterlet
  workloadSelectorTemplates:
  - "k8s:sa:cluster5-work-sa"
```

If the management cluster is in the hub's trust domain, use that trust domain and leave out
`federatesWith`. More on entries: [`../clusterspiffeid.md`](../clusterspiffeid.md).

### 4.4 Accept

```shell
clusteradm accept --clusters cluster5
```

The agent created `cluster5` when it joined, and the hub pinned it to the management cluster's
trust domain ([README 6.3](README.md#63-accept-a-cluster)).

## 5. The admission policies

Split across the two clusters, as in `Hosted` mode.

| Policy | Name | On | What it does here |
| --- | --- | --- | --- |
| Agent namespace | `ocm-spiffe-agents-cluster5` | management cluster | In the namespace `cluster5`, only the operator changes `cluster5-agent`. A pod with role `klusterlet` must use `cluster5-work-sa`. No `exec`, `attach`, debug container or image change. |
| Add-on namespaces | `ocm-spiffe-addons-cluster5` | managed cluster | In namespaces labeled `open-cluster-management.io/spiffe-addon-namespace: cluster5`, only `cluster5`'s agent (as the work agent) creates workloads. |
| Namespace label | `ocm-spiffe-addon-namespace-label` | managed cluster | The add-on label can be set only when a namespace is created. |

Recovery after a policy change covers only this Klusterlet. A change on the management cluster is
seen within seconds; a change on the managed cluster at the next resync, within five minutes
(`questions.md` Q20).

## 6. The lifecycle

| Step | What happens in `SingletonHosted` mode | More |
| --- | --- | --- |
| Send work | The agent on the management cluster watches `cluster5`'s `ManifestWork`s with its `klusterlet` SVID and applies them on the **managed** cluster. The hub sees user `...:cluster5:klusterlet` for both registration and work. | [README 6.4](README.md#64-send-work-to-a-cluster) |
| Renew | SPIRE on the management cluster renews the one SVID; registration and work reconnect together. | [README 6.5](README.md#65-renew-an-svid) |
| Deny | As in `Singleton`: the connections are closed, but the gate does not refuse the agent afterwards, because it is also the registration agent. See section 7. | [README 6.6](README.md#66-deny-a-cluster-or-disable-an-add-on) |
| Delete | Remove the one entry on the management cluster, then delete the `ManagedCluster`. The hub holds the name for `maxSVIDLifetime + maxClockSkew`. | [README 6.7](README.md#67-delete-a-cluster) |
| Add-ons | As in `Hosted`: an add-on deployed the usual way runs on the **managed** cluster, in a namespace labeled `cluster5`, so the managed cluster needs a SPIRE agent, the CSI driver and the add-on's entry. | [README 6.8](README.md#68-run-an-add-on) |
| Remove the Klusterlet | The operator deletes its policies and bindings on both clusters, and the shared label policy if no other Klusterlet uses `spiffe`. | [README 6.14](README.md#614-move-an-existing-cluster-to-spiffe-and-back) |

## 7. Known limits

This mode has the limits of both `Singleton` and `Hosted`.

| Limit | From | Details |
| --- | --- | --- |
| **A denied cluster can still get work** | `Singleton` | The one agent's role covers registration and work, so the gate lets it through while the cluster is denied, and the kept work RoleBinding lets its work requests pass (`questions.md` Q19). Use `Hosted` mode if this matters. |
| **The managed cluster's kubeconfig is a Secret** | `Hosted` | `external-managed-kubeconfig` on the management cluster is long-lived. Not a hub credential, and every driver needs it; whether it is in scope is open (`questions.md` Q15). |
| Policies on the managed cluster | `Hosted` | Checked only at the five-minute resync (`questions.md` Q20). |
| Short SVIDs | all | With SVIDs under 10 minutes, an update can arrive up to about a minute late after a reconnect (`questions.md` Q18). |

## 8. Test status

e2e on kind with SPIRE 1.15.5 ([`../e2e-results.md`](../e2e-results.md)): cluster `cluster5`, agent
on `ocm-e2e-spoke` (management, shared with `cluster4` in `Hosted` mode), workloads on
`ocm-e2e-managed2`. The managed cluster had no SPIRE.

| Check | Result |
| --- | --- |
| Join with no token, entry after `SPIFFEAdmissionReady`, accept | pass |
| No hub credential Secret | pass |
| Cluster pinned to the trust domain | pass |
| `ManifestWork` applied on the managed cluster | pass |
| Left alone while `cluster4`'s policy was tampered with and recovered | pass |
| `exec` into the agent refused | not run (the same policy passed in `Hosted` and `Singleton`) |
| Add-ons | not run (the managed clusters had no SPIRE) |
| Rotation, deny, delete | run in `Default` mode only; the hub side is the same |

Unit tests for this mode: the `SingletonHosted` case in
[`klusterlet_spiffe_test.go`](../../ocm/pkg/operator/operators/klusterlet/controllers/klusterletcontroller/klusterlet_spiffe_test.go).
