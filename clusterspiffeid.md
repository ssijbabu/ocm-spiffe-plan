# Creating the ClusterSPIFFEIDs

How to give the hub gRPC server, the Klusterlet agents and add-on agents their SPIFFE IDs with the
SPIRE controller manager's `ClusterSPIFFEID`. OCM never creates these objects: the platform does,
and OCM only checks the IDs it receives. Used by Days 14, 18, 22 and 26–28.

**Design source:** proposal, "Hub setup", "Managed cluster setup" for each install mode, "Add-ons",
"Trust domain pinning", "Risks and Mitigation".

## How it works

1. The SPIRE controller manager runs next to the SPIRE server and watches `ClusterSPIFFEID`s and
   pods in its cluster.
2. For each pod that matches a `ClusterSPIFFEID`, it creates a SPIRE registration entry with the
   rendered `spiffeIDTemplate`, parented to the node the pod runs on.
3. The pod opens the Workload API through the `csi.spiffe.io` volume. The SPIRE agent attests it
   (namespace, ServiceAccount, labels) and hands it the SVID, in memory.

The layout of the IDs is set by the hub's agent ID templates (`agentIDTemplate`,
`addOnAgentIDTemplate`; `questions.md` Q7). The hub reads the cluster, role, add-on and agent from
the ID with them, so an entry whose `spiffeIDTemplate` doesn't follow them gets an SVID the hub
refuses. The examples below use the default templates, which are the proposal's scheme; see
[Custom ID templates](#custom-id-templates) for other layouts. The Klusterlet lists the exact IDs
its agents expect in `status.spiffe.agentIDs`, and `clusteradm join` prints them.

## Before you start

| Check | How | Why |
| --- | --- | --- |
| Controller manager is running | `kubectl -n spire-server get pod spire-server-0 -o jsonpath='{.spec.containers[*].name}'` lists `spire-controller-manager` | Nothing reconciles the objects otherwise |
| Its class name | default is `<release namespace>-<release name>`: `spire-server-spire` on ocm-hub | The chart sets `watchClassless: false`, so a `ClusterSPIFFEID` without `spec.className` is **ignored** |
| CSI driver is installed | `kubectl get csidriver csi.spiffe.io` | Agents get the socket through it |
| The pods' cluster has a controller manager that can reach the SPIRE server | see [Topology](#topology) | `ClusterSPIFFEID` only selects pods in the cluster where the controller manager runs |
| The chart's `default` ClusterSPIFFEID | `kubectl get clusterspiffeid` | It gives **every** pod `spiffe://<td>/ns/<ns>/sa/<sa>` too. That is fine: OCM picks its SVID by ID, never by position (Day 7), but the agent then holds two SVIDs |

## Rules for every entry

- **Select on all three:** namespace (`namespaceSelector`), the `open-cluster-management.io/spiffe-role`
  label (`podSelector`) and the ServiceAccount (`workloadSelectorTemplates: ["k8s:sa:<sa>"]`). The
  role label alone can be set by anyone who can create a pod; the admission policy (Days 19–20)
  pins label and SA together.
- **One cluster name per entry.** Never template the cluster name, role, add-on or agent name
  from a pod label or namespace that a user could choose. Only a `{namespace}` segment may come
  from `{{ .PodMeta.Namespace }}`, because the hub never trusts it.
- **`ttl` at most `maxSVIDLifetime`** (ClusterManager, default `24h`). The delete guard relies on
  it. Leave `ttl` unset to use the server's default X.509 TTL, and check that default is not
  longer either.
- **Other trust domains:** set `federatesWith: [ocm-hub.example.org]` so the agent also gets the
  hub's bundle, and add the agent's trust domain to the hub's `agentTrustDomains`.
- **Order on a managed cluster:** apply agent entries only after the Klusterlet reports
  `SPIFFEAdmissionReady` (Day 21). Before that, any pod with the right label and SA could get the
  agent's SVID.
- **Deleting a cluster:** delete its `ClusterSPIFFEID`s first, then the `ManagedCluster` (Day 13).

## Which entries to create

With the default templates:

| Workload | Mode | Where | Namespace | `spiffe-role` | ServiceAccount | SPIFFE ID |
| --- | --- | --- | --- | --- | --- | --- |
| Hub gRPC server | all | hub | `open-cluster-management-hub` | none; `app: cluster-manager-grpc-server` | `grpc-server-sa` | `spiffe://<hub td>/ocm/hub/grpc-server` |
| Registration agent | `Default` | managed cluster | `open-cluster-management-agent` | `registration` | `klusterlet-registration-sa` | `spiffe://<td>/ocm/cluster/<cluster>/agent/registration` |
| Work agent | `Default` | managed cluster | `open-cluster-management-agent` | `work` | `klusterlet-work-sa` | `spiffe://<td>/ocm/cluster/<cluster>/agent/work` |
| Klusterlet agent | `Singleton` | managed cluster | `open-cluster-management-agent` | `klusterlet` | `klusterlet-work-sa` | `spiffe://<td>/ocm/cluster/<cluster>/agent/klusterlet` |
| Registration agent | `Hosted` | management cluster | `<klusterlet>` | `registration` | `<klusterlet>-registration-sa` | `spiffe://<mgmt td>/ocm/cluster/<cluster>/agent/registration` |
| Work agent | `Hosted` | management cluster | `<klusterlet>` | `work` | `<klusterlet>-work-sa` | `spiffe://<mgmt td>/ocm/cluster/<cluster>/agent/work` |
| Klusterlet agent | `SingletonHosted` | management cluster | `<klusterlet>` | `klusterlet` | `<klusterlet>-work-sa` | `spiffe://<mgmt td>/ocm/cluster/<cluster>/agent/klusterlet` |
| Add-on agent | all | where the add-on runs | the add-on's namespace | `addon` | the add-on's SA | `spiffe://<td>/ocm/cluster/<cluster>/addon/<addon>/agent/<agent>` |

`<td>` is the trust domain of the SPIRE that issues the SVID. `Singleton` has no `registration` or
`work` entries; `Default` has no `klusterlet` entry. In hosted modes, the agents run on the
management cluster, so the entries go there and use its trust domain.

## Examples

All examples use `className: spire-server-spire` (ocm-hub's release). Use your own release's class.

### Hub gRPC server

On the hub. `federatesWith` only when agents are in other trust domains.

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-hub-grpc-server
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/hub/grpc-server"
  federatesWith:
  - east.example.org
  - west.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-hub
  podSelector:
    matchLabels:
      app: cluster-manager-grpc-server
  workloadSelectorTemplates:
  - "k8s:sa:grpc-server-sa"
```

### Default mode: cluster1, in the hub's trust domain

Two entries. No `federatesWith`: same trust domain.

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

### Default mode: cluster2, in east.example.org

Same shape, east's trust domain, federated with the hub's. The hub must list `east.example.org`
in `agentTrustDomains`.

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster2-work
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster2/agent/work"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: work
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-work-sa"
# ocm-cluster2-registration: the same with agent/registration, role registration,
# klusterlet-registration-sa
```

### Singleton mode: cluster1

One entry. It reuses `klusterlet-work-sa`; the role label tells it apart from a work agent.

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

### Hosted mode: cluster4 on management cluster mgmt1 (east.example.org)

On mgmt1. The namespace is the Klusterlet's name, which is what keeps cluster5's pods from
getting cluster4's IDs. One pair of entries per hosted cluster.

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

### SingletonHosted mode: cluster5 on mgmt1

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

### Add-on: my-addon on cluster2

On the cluster where the add-on runs. The add-on framework adds `spiffe-role: addon` and the CSI
volume to the add-on's workloads (Day 23).

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster2-my-addon
spec:
  className: spire-server-spire
  spiffeIDTemplate: "spiffe://east.example.org/ocm/cluster/cluster2/addon/my-addon/agent/my-addon-agent"
  federatesWith:
  - ocm-hub.example.org
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent-addon
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: addon
  workloadSelectorTemplates:
  - "k8s:sa:my-addon-agent-sa"
```

## Custom ID templates

If the hub and the Klusterlets use other templates, the entries follow them. Example, with
`agentIDTemplate: spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}` on the
ClusterManager and the Klusterlet:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster1-work
spec:
  className: spire-server-spire
  # cluster and role fixed; only the namespace comes from the pod
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/k8s/cluster1/ns/{{ .PodMeta.Namespace }}/ocm/work"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: work
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-work-sa"
```

| Template placeholder | Value in the entry | Must equal |
| --- | --- | --- |
| `{trustDomain}` | the issuing SPIRE's trust domain | the Klusterlet's `spiffe.trustDomain`, and listed in `agentTrustDomains` |
| `{cluster}` | fixed text, `cluster1` | the Klusterlet's `clusterName` |
| `{role}` | fixed text, `registration`, `work` or `klusterlet` | the pod's `spiffe-role` label |
| `{namespace}` | fixed text or `{{ .PodMeta.Namespace }}` | the namespace the agent pods run in (the Klusterlet's namespace on the management cluster in hosted modes) |
| `{addon}`, `{agent}` (add-on template) | fixed text | the add-on's name and agent name |

Changing a template on a running hub refuses every agent until its Klusterlet and entries follow.
Order: add the new entries (agents then hold both SVIDs), change the ClusterManager, change each
Klusterlet, then delete the old entries.

## Topology

A `ClusterSPIFFEID` only matches pods in the cluster where its controller manager runs, and that
controller manager writes entries through its local SPIRE server's API.

| Topology | Where managed-cluster entries go |
| --- | --- |
| A SPIRE server per cluster (own trust domain, federated; or nested under the hub's) | `ClusterSPIFFEID` on that cluster, as above. Fits directly. |
| One SPIRE server on the hub, managed cluster runs only SPIRE agents (Day 26's e2e choice) | The managed cluster has **no** controller manager, and the hub's one only sees hub pods. Create the entries with `spire-server entry create` on the hub server, parented to the managed cluster's agents and with selectors `k8s:ns:…`, `k8s:sa:…`, `k8s:pod-label:open-cluster-management.io/spiffe-role:…`. Day 26 Task 3 assumes a controller manager on the spoke; fix that or pick a per-cluster server. |

Decide this with Open Question 4 (`questions.md`).

## Check

```shell
# The controller manager picked it up (no className → no status).
kubectl get clusterspiffeid ocm-cluster1-work -o jsonpath='{.status.stats}'
# Expect podsSelected ≥ 1, entryFailures 0 once the agent pod runs.

# The entry exists on the SPIRE server.
kubectl -n spire-server exec spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show \
  -spiffeID spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work
```

Then the smoke-test pod from Day 26 Task 4 should print the expected SPIFFE ID.

| Symptom | Likely cause |
| --- | --- |
| No `status` on the object | Missing or wrong `className` |
| `podsSelected: 0` | Namespace or role label doesn't match; agent pod not running yet |
| Entry exists, pod gets only the `ns/…/sa/…` SVID | `k8s:sa:` selector doesn't match the pod's ServiceAccount |
| Agent gets its SVID, TLS to the hub fails | Missing `federatesWith` on one side, or no `ClusterFederatedTrustDomain` |
| TLS works, hub refuses the agent with `matches neither agentIDTemplate` | The entry's ID, the Klusterlet's template and the hub's template don't all agree |
| TLS works, hub refuses the agent otherwise | Trust domain not in `agentTrustDomains`, or cluster pinned to another domain |
| Agent not ready, `ErrSVIDNotFound` listing other IDs | The entry's ID differs from `status.spiffe.agentIDs` |
| gRPC server not ready, `ErrSVIDNotFound` in the log | Hub entry missing or ID differs from `serverSPIFFEID` |
