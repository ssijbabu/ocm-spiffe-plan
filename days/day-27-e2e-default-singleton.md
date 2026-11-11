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
