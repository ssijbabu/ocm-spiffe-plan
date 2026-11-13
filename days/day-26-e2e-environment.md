# Day 26: e2e environment with kind and SPIRE

**Goal:** a script that creates a hub and a managed cluster in kind, installs SPIRE in one trust
domain, and gets an SVID into a test pod on each cluster. This is the base for Days 27 and 28.

**Repo:** `ocm`, branch `spire`, new `test/e2e/spiffe/` (script and values), or `hack/` if the repo
keeps e2e setup there.

## Before you start

| Input | Where |
| --- | --- |
| SPIRE charts | `https://spiffe.github.io/helm-charts-hardened/`: `spire-crds` 0.6.1, `spire` 0.30.4 (SPIRE 1.15.5), the same as ocm-hub |
| Existing e2e | `test/e2e` and how `make test-e2e` creates kind clusters |
| Decision needed | proposal Open Question 4: the reference SPIRE topology |

## Task 1: decide the topology

| Option | How the spoke's SPIRE agent gets identities | Fits e2e? |
| --- | --- | --- |
| One SPIRE server on the hub, spoke agents attest to it | spoke agent uses `k8s_psat`; the hub server must validate spoke tokens, so it needs access to the spoke's API (kubeconfig per cluster) | yes, simplest single trust domain |
| Nested: a SPIRE server per cluster, downstream of the hub's | each spoke server gets an intermediate CA from the hub server | closer to production at scale; more setup |
| Federation, one trust domain per cluster | multiple trust domains | not in this proposal's single-trust-domain design |

Recommendation for e2e: **one SPIRE server on the hub**. Write the decision in
`plan/questions.md` (it answers OQ4 for the tests) and in the e2e README.

## Task 2: the script

**New file:** `test/e2e/spiffe/setup.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
TD=ocm-hub.example.org
kind create cluster --name ocm-hub   --image kindest/node:v1.32.0
kind create cluster --name ocm-spoke --image kindest/node:v1.32.0

helm repo add spiffe https://spiffe.github.io/helm-charts-hardened/ && helm repo update

# Hub: SPIRE server + agent + CSI driver + controller manager.
helm --kube-context kind-ocm-hub upgrade --install spire-crds spiffe/spire-crds \
  --version 0.6.1 -n spire-server --create-namespace
helm --kube-context kind-ocm-hub upgrade --install spire spiffe/spire \
  --version 0.30.4 -n spire-server -f test/e2e/spiffe/values-hub.yaml \
  --set global.spire.trustDomain=$TD --set global.spire.clusterName=ocm-hub

# Spoke: SPIRE agent + CSI driver + controller manager, pointing at the hub's server.
HUB_SPIRE=$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' ocm-hub-control-plane)
helm --kube-context kind-ocm-spoke upgrade --install spire-crds spiffe/spire-crds \
  --version 0.6.1 -n spire-server --create-namespace
helm --kube-context kind-ocm-spoke upgrade --install spire spiffe/spire \
  --version 0.30.4 -n spire-server -f test/e2e/spiffe/values-spoke.yaml \
  --set global.spire.trustDomain=$TD --set global.spire.clusterName=ocm-spoke \
  --set "spire-agent.server.address=$HUB_SPIRE"
```

**Files:** `values-hub.yaml` and `values-spoke.yaml`. Write them from the chart's `values.yaml` for
0.30.4. Things to set:

- hub: expose the SPIRE server on a NodePort so the spoke agent can reach it;
- hub: the server's `k8s_psat` attestor knows both clusters (`ocm-hub`, and `ocm-spoke` with the
  spoke's kubeconfig in a Secret the server mounts);
- spoke: `spire-server.enabled: false`, agent pointed at the hub server, CSI driver on;
- both: JWT/X.509 key type as on ocm-hub (`rsa-2048` for JWT; not needed here, but keep the same).

Check every key name against the chart before using it; chart keys change between versions.

## Task 3: entries for the test

**New file:** `test/e2e/spiffe/entries.yaml`, applied with the SPIRE controller manager. The spoke
cluster's entries are created on the spoke (its controller manager registers them with the hub
server under the spoke's cluster name).

```yaml
# on ocm-hub
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-hub-grpc-server
spec:
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/hub/grpc-server"
  namespaceSelector:
    matchLabels: {kubernetes.io/metadata.name: open-cluster-management-hub}
  podSelector:
    matchLabels: {app: cluster-manager-grpc-server}
  workloadSelectorTemplates: ["k8s:sa:grpc-server-sa"]
---
# on ocm-spoke (cluster1, Default mode)
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster1-registration
spec:
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/registration"
  namespaceSelector:
    matchLabels: {kubernetes.io/metadata.name: open-cluster-management-agent}
  podSelector:
    matchLabels: {open-cluster-management.io/spiffe-role: registration}
  workloadSelectorTemplates: ["k8s:sa:klusterlet-registration-sa"]
# ... ocm-cluster1-work (role work, klusterlet-work-sa)
# ... ocm-cluster1-klusterlet (role klusterlet, klusterlet-work-sa) for Singleton
# ... ocm-my-addon (add-on namespace label + role addon + my-addon-agent-sa)
```

Remember the order from the proposal: spoke entries are applied only after the Klusterlet reports
`SPIFFEAdmissionReady` (Day 21). The e2e script must follow that order, not apply everything at
once.

## Task 4: smoke test

**Input:** a throwaway pod on the spoke that matches the work entry.

```shell
kubectl --context kind-ocm-spoke create ns open-cluster-management-agent
kubectl --context kind-ocm-spoke -n open-cluster-management-agent create sa klusterlet-work-sa
kubectl --context kind-ocm-spoke -n open-cluster-management-agent apply -f test/e2e/spiffe/svid-check-pod.yaml
kubectl --context kind-ocm-spoke -n open-cluster-management-agent logs svid-check
```

`svid-check-pod.yaml` runs `ghcr.io/spiffe/spire-agent:1.15.5` with
`api fetch x509 -socketPath /spiffe-workload-api/spire-agent.sock`, the role label `work`, SA
`klusterlet-work-sa` and the CSI volume.

**Output:**

```text
Received 1 svid after 25.4ms

SPIFFE ID:		spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work
SVID Valid After:	2026-11-26 10:00:02 +0000 UTC
SVID Valid Until:	2026-11-26 11:00:12 +0000 UTC
CA #1 Valid After:	2026-11-26 09:58:40 +0000 UTC
CA #1 Valid Until:	2026-11-27 09:58:50 +0000 UTC
```

Delete the namespace afterwards; the real agents come on Day 27.

## Task 5: commit

```shell
git add test/e2e/spiffe && git commit -s -m ":seedling: kind + SPIRE environment for SPIFFE e2e"
```

## End of day

| Output | Check |
| --- | --- |
| `setup.sh`, values files, entries | a fresh run ends with the smoke test printing the expected SPIFFE ID |
| Topology decision | written down |

## Update: multiple agent trust domains

For the multi-trust-domain case, run the spoke's SPIRE in its own trust domain (`east.example.org`) and federate it with the hub's (`ClusterFederatedTrustDomain` on both sides, `federatesWith` in the `ClusterSPIFFEID`s). Keep one same-domain spoke as well.

## Update: agent ID templates

Write `entries.yaml` from `plan/clusterspiffeid.md` (it has every entry, per mode). Two additions:

- Every `ClusterSPIFFEID` needs `spec.className` matching the chart's controller manager
  (`<release namespace>-<release name>`, `spire-server-spire` here); the chart ignores objects
  without it. Check `controllerManager.className` in the 0.30.4 values.
- Add a second set of agent entries for the custom template used on Day 27, Task 7:
  `spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}`. Keep the cluster name and role
  fixed in each entry; only the namespace segment may use `{{ .PodMeta.Namespace }}`, because the
  hub never trusts it:

  ```yaml
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/k8s/cluster1/ns/{{ .PodMeta.Namespace }}/ocm/work"
  ```

  Apply them only on Day 27, Task 7.

Smoke test: run the `svid-check` pod once more with both sets applied. It should receive both IDs,
which is also the input for checking that agents pick by ID.
