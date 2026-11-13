# Day 28: e2e, Hosted and SingletonHosted

**Goal:** the same lifecycle works when the agents run on a management cluster, including two
Klusterlets on one management cluster that don't affect each other.

**Repo:** `ocm`, branch `spire`, `test/e2e`.

## Before you start

| Input | Where |
| --- | --- |
| Environment | Day 26, plus a third kind cluster `ocm-mgmt` with a SPIRE agent attesting to the hub server |
| Hosted entries | proposal: entries on the management cluster select on the Klusterlet namespace and role label |

Topology for today:

```text
ocm-hub      hub, SPIRE server
ocm-mgmt     management cluster: Klusterlets cluster4 and cluster5 (agents run here)
ocm-spoke    managed cluster cluster4 (workloads land here)
ocm-spoke2   managed cluster cluster5
```

## Task 1: entries on the management cluster

Hosted agents run in the namespace named after the Klusterlet. For cluster4:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-cluster4-work
spec:
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/cluster/cluster4/agent/work"
  namespaceSelector:
    matchLabels: {kubernetes.io/metadata.name: cluster4}
  podSelector:
    matchLabels: {open-cluster-management.io/spiffe-role: work}
  workloadSelectorTemplates: ["k8s:sa:cluster4-work-sa"]
# + ocm-cluster4-registration (role registration, cluster4-registration-sa)
# + the same pair for cluster5 in namespace cluster5
```

## Task 2: join in hosted mode

```shell
clusteradm --context kind-ocm-mgmt join --cluster-name cluster4 --registration-auth spiffe \
  --hub-grpc-endpoint <addr> --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server \
  --mode hosted --managed-cluster-kubeconfig ./ocm-spoke.kubeconfig
```

**Output on the management cluster:**

```text
$ kubectl --context kind-ocm-mgmt -n cluster4 get deploy
NAME                          READY
cluster4-registration-agent   1/1
cluster4-work-agent           1/1
```

`external-managed-kubeconfig` exists in `cluster4` (Open Question 5; expected). No hub credential
Secret exists.

## Task 3: specs

Run Day 27's specs 1 to 9 for cluster4, with these differences:

| Spec | Hosted difference |
| --- | --- |
| Work | the ConfigMap lands on `ocm-spoke`, not on `ocm-mgmt` |
| exec blocked | exec into `cluster4-work-agent` on `ocm-mgmt` is refused |
| Add-on | a hosted add-on runs on `ocm-mgmt` in an add-on namespace labeled `cluster4`; its SVID is `.../cluster/cluster4/addon/my-addon/agent/my-addon-agent` |

**Two Klusterlets on one management cluster:**

| # | Step | Expected |
| --- | --- | --- |
| 10 | `cluster5`'s work agent identity creates a Deployment in `cluster4`'s add-on namespace (impersonate it) | refused: `only this Klusterlet's work agent may create or change add-on workloads` |
| 11 | delete `cluster4`'s agent policy (forces recovery for cluster4) | cluster4 agents scale to 0 and back; cluster5's agents and add-ons keep running (pod restart count unchanged) |
| 12 | label `default` on `ocm-mgmt` with `spiffe-addon-namespace: cluster4` during the gap from #11 | nothing in `default` deleted; label removed again by the operator |

## Task 4: SingletonHosted

Join cluster4 again with `--mode hosted --singleton`. Expected: one Deployment `cluster4-agent`,
label `klusterlet`, SA `cluster4-work-sa`. Run specs 1 to 9.

## Task 5: run and record

```shell
dgo 'go test ./test/e2e/... -run "SPIFFE.*Hosted" -timeout 120m -v' 2>&1 | tee ../plan/e2e-day28.log | grep -E '^(--- |ok|FAIL)'
```

Add two columns (`Hosted`, `SingletonHosted`) to `plan/e2e-results.md`, and rows 10 to 12.

## Task 6: commit and push

```shell
git add -A && git commit -s -m ":white_check_mark: e2e for SPIFFE registration in hosted modes"
git push origin spire
```

## Update: agent ID templates

Leave the hub on the custom template from Day 27, Task 7. The hub accepts one template at a time,
so cluster4 (Hosted) and cluster5 (SingletonHosted) use it from the start, on their Klusterlets
and in their entries on mgmt1. Their `{namespace}` is the Klusterlet's namespace on the
management cluster: the operator renders
`spiffe://east.example.org/k8s/cluster4/ns/cluster4/ocm/registration` and
`spiffe://east.example.org/k8s/cluster5/ns/cluster5/ocm/klusterlet`. This checks that
`{namespace}` is the namespace the agent pods actually run in, in hosted mode.

## Status (2026-11-13)

Done, committed locally; results and gaps in `e2e-results.md`.

## In simple words: what changed and why

Day 28 ran the same checks for the two hosted modes. In these modes the agents don't run on the
cluster they manage. They run on a separate management cluster, in a namespace named after the
Klusterlet. For the test, the managed cluster of Day 27 became the management cluster, and two
more kind clusters were added as managed clusters. Those two have no SPIRE at all, because no
agent runs on them.

What passed in `Hosted` and `SingletonHosted`: join with no token, no Secret with a hub
credential, pinning to the trust domain, and a `ManifestWork` whose content lands on the managed
cluster and not on the management cluster. In `Hosted` mode, running a command inside an agent
pod is refused.

The check that matters most for hosted modes is that Klusterlets on one management cluster don't
affect each other. We deleted one Klusterlet's agent policy. At first nothing happened until the
operator's five-minute resync, which is a window where a fake agent pod could get an SVID. After
the fix (the operator now watches the policies), the policy was back and the agents were replaced
within six seconds, and the other Klusterlet on the same management cluster was not touched.

Three things are left open. Two planned checks (one Klusterlet's work agent trying to use
another's add-on namespace, and the label put on `default` during recovery) were not run on real
clusters. The second has a unit test from Day 21. For the first, the tests of Day 20 only show
that an ordinary user is refused by the same rule, not another Klusterlet's work agent. The custom
ID template was not run either. And in hosted modes a Secret with the kubeconfig of the managed cluster still exists on
the management cluster. It is not a hub credential and every driver needs it, but whether that is
acceptable has to be confirmed and written in the proposal (`questions.md` Q15).
