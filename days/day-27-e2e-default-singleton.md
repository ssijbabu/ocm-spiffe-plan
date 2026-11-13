# Day 27: e2e, Default and Singleton

**Goal:** the full lifecycle works on real clusters with real SPIRE in `Default` and `Singleton`
modes: join, accept, work, add-on, rotation, revocation.

**Repo:** `ocm`, branch `spire`, `test/e2e`.

## Before you start

| Input | Where |
| --- | --- |
| Environment | Day 26, `test/e2e/spiffe/setup.sh` |
| Images | build ocm images from `spire` and load them into kind: `make images`, then `kind load docker-image` (check the Makefile targets) |
| clusteradm | build from `clusteradm` `spire` (Day 24) |

## Task 1: install OCM

```shell
CTX_HUB=kind-ocm-hub CTX_SPOKE=kind-ocm-spoke
clusteradm --context $CTX_HUB init --registration-drivers spiffe --spiffe-trust-domain ocm-hub.example.org \
  --spiffe-endpoint-type hostname --bundle-version=<local images>
kubectl --context $CTX_HUB -n open-cluster-management-hub get pods
```

**Output:**

```text
NAME                                                     READY   STATUS
cluster-manager-grpc-server-6c8d9b7f5-tq2lm              1/1     Running
cluster-manager-registration-controller-...              1/1     Running
...
```

If the gRPC server isn't ready, check its log for `ErrSVIDNotFound`: the hub's `ClusterSPIFFEID`
is missing or doesn't match.

## Task 2: join and accept (Default)

```shell
clusteradm --context $CTX_SPOKE join --cluster-name cluster1 --registration-auth spiffe \
  --hub-grpc-endpoint <hub node IP>:<grpc-spiffe nodeport> \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server
# join waits for SPIFFEAdmissionReady, then:
kubectl --context $CTX_SPOKE apply -f test/e2e/spiffe/entries-cluster1-default.yaml
clusteradm --context $CTX_HUB accept --clusters cluster1
kubectl --context $CTX_HUB get managedcluster cluster1
```

**Output:**

```text
NAME       HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
cluster1   true                                  True     True        40s
```

**Check there are no credentials on the spoke:**

```shell
kubectl --context $CTX_SPOKE -n open-cluster-management-agent get secret
```

Expected: no `bootstrap-hub-kubeconfig`, no `hub-kubeconfig-secret`.

## Task 3: the e2e test suite

Write these as Ginkgo specs in `test/e2e/spiffe_test.go`, run against the two contexts.

| # | Step | Input | Expected |
| --- | --- | --- | --- |
| 1 | Work | ManifestWork with ConfigMap `e2e-cm` in `default` | `e2e-cm` on the spoke; work `Applied=True`, `Available=True` |
| 2 | Status feedback | feedback rule on `e2e-cm` | value appears in the work status |
| 3 | Add-on enable | `clusteradm addon enable --names my-addon --clusters cluster1` (a test add-on using the Day 23 helper) | MCA status `kubeClient.driver: spiffe`, `RegistrationApplied=True`, `Available=True` |
| 4 | Rotation under load | SPIRE entry TTL set to 5m; apply 1 ManifestWork every 10s for 15m | no failed applies; hub log shows connections closed at expiry and reopened |
| 5 | Agent down past expiry | scale the work agent to 0 for 10m (longer than the TTL), then back to 1 | it gets a fresh SVID and works again; no manual step |
| 6 | Add-on revoke | delete the `my-addon` MCA | add-on connection closed (hub log); work agent unaffected |
| 7 | Cluster deny | `hubAcceptsClient: false` | lease stops; ManifestWork updates stop reaching the spoke |
| 8 | Cluster delete | remove cluster1's `ClusterSPIFFEID`s, then delete the ManagedCluster | stays terminating for `maxSVIDLifetime + maxClockSkew` (set to 10m + 1m for the test), then gone; not recreated |
| 9 | exec blocked | `kubectl exec` into the work agent pod as cluster-admin | refused with the policy message |

Example for #9:

```shell
kubectl --context $CTX_SPOKE -n open-cluster-management-agent exec deploy/klusterlet-work-agent -- true
```

```text
Error from server (Forbidden): ... ValidatingAdmissionPolicy 'ocm-spiffe-agents-klusterlet' with binding
'ocm-spiffe-agents-klusterlet' denied request: exec and attach are not allowed on OCM agent pods; read the logs instead
```

## Task 4: Singleton

Unjoin, then join again with `--singleton` and the singleton entry (role `klusterlet`,
`klusterlet-work-sa`). Run specs 1 to 9 again.

| Extra check | Expected |
| --- | --- |
| Agent pods | one Deployment `klusterlet-agent`, label `spiffe-role: klusterlet` |
| Hub sees | user `system:open-cluster-management:cluster1:klusterlet` for both registration and work requests |

## Task 5: run and record

```shell
dgo 'go test ./test/e2e/... -run SPIFFE -timeout 90m -v' 2>&1 | tee ../plan/e2e-day27.log | grep -E '^(--- |ok|FAIL|•)'
```

**Output:** all specs pass in both modes. Put the summary in `plan/e2e-results.md`:

```markdown
| Spec | Default | Singleton |
| --- | --- | --- |
| Work | pass | pass |
| ...
```

## Task 6: commit

```shell
git add -A && git commit -s -m ":white_check_mark: e2e for SPIFFE registration in Default and Singleton modes"
```

## Update: multiple agent trust domains

Add rows: cluster2 in `east.example.org` joins and works; its pinned annotation is set; an SVID for cluster2 from `west.example.org` is refused; removing `east.example.org` from `agentTrustDomains` disconnects cluster2.

## Update: agent ID templates

### Task 7: switch to a custom template

After the default-template flows pass, switch cluster1 (Default mode) to
`spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}`:

1. Apply the custom-template entries from Day 26. The agents now hold both SVIDs and keep using
   the default one.
2. Set `agentIDTemplate` on the ClusterManager. The hub now refuses the default IDs: the agents'
   connections drop and their reconnects fail with `matches neither agentIDTemplate`, visible in
   the Klusterlet's `HubConnectionDegraded` condition.
3. Set the same `agentIDTemplate` on the Klusterlet. The operator re-renders `--spiffe-agent-id`;
   the agents restart, pick the new SVID and reconnect.
4. Check: the ManagedCluster stays `Available`, its ManifestWorks and add-ons are untouched, and
   `status.spiffe.agentIDs` shows the new IDs.
5. Delete the default-layout entries.

**Expected:** connections drop between steps 2 and 3, and recover by themselves after step 3.
Record how long the gap was. The hub accepts one template at a time, so changing it on a live hub
interrupts every cluster until its Klusterlet matches; see `questions.md` Q7.

Also run one add-on (enable, status sync) with a custom `addOnAgentIDTemplate` on both sides.

## Update: what Days 4–9 built (2026-11-13)

Expect a 5–10s gap in event delivery after each SVID expiry (the base client's first reconnect
delay, `questions.md` Q9). "SVID rotation under load" should check that nothing is lost after
the gap (the resync covers it), not that there is no gap.

## Status (2026-11-13)

Done, committed locally; results and gaps in `e2e-results.md`.

## In simple words: what changed and why

Day 27 ran the whole thing on the real clusters from Day 26, in `Default` and `Singleton` modes.
Two more scripts build the OCM images from our branch and install OCM: the hub with only the
`spiffe` driver, a join with no token, the SPIRE entries applied only after the operator reports
`SPIFFEAdmissionReady`, and then accept. kind has no load balancer, so a NodePort stands in for
it.

What passed: the cluster joins with no token; no Secret with a hub credential exists on the
managed cluster; the cluster is pinned to its trust domain; a `ManifestWork` is applied; a
command run inside an agent pod is refused, even for a cluster admin; and deleting a Klusterlet
removes its policies. In `Default` mode we also ran the longer checks. With SVIDs that live two
minutes and updates sent for five minutes, nothing was lost, though one update arrived 42 seconds
late (`questions.md` Q18; it doesn't happen with one-hour SVIDs). A denied cluster stops
receiving work. A deleted cluster is held for the configured time and then released on time.

Not everything in the plan was run. Status feedback, enabling and removing an add-on (this needs
a test add-on image built on the new helper), and the switch to a custom ID template are still
open. The results table is in `e2e-results.md`.

The main value of this day was the bugs. Real SPIRE and real clusters found problems that the
unit and integration tests had not: the lost recovery record and the health check that killed the
work agent (Days 17 and 21), the denied cluster whose work agent kept working (Day 12), the
policies left behind after a Klusterlet was deleted (Day 21), and the wrong order in the
`clusteradm init` message (Day 24). All are fixed and have unit tests.
