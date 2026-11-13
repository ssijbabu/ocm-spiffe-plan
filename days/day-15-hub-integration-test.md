# Day 15: hub integration test

**Goal:** an integration test that runs the hub (registration controllers plus the SPIFFE gRPC
listener) against envtest, with a fake agent that uses only an SVID from the fake Workload API. It
covers register, accept, rotation, deny and delete.

**Repo:** `ocm`, branch `spire`, `test/integration/registration`.

## Before you start

| Input | Where |
| --- | --- |
| Existing gRPC test | `test/integration/registration/spokecluster_grpc_test.go` (copy its structure) |
| Suite setup | `test/integration/registration/integration_suite_test.go` |
| Fake Workload API | sdk-go `pkg/testing/fakespire` (Day 6) |

## Task 1: suite setup for SPIFFE

**Change** `integration_suite_test.go`: in `BeforeSuite`, if not already present, start:

1. a `fakespire.CA` for `ocm-hub.example.org`;
2. a fake Workload API for the hub, serving `spiffe://ocm-hub.example.org/ocm/hub/grpc-server`
   with a 1h TTL;
3. the gRPC server with `--grpc-enabled=false --spiffe-enabled --spiffe-trust-domain=ocm-hub.example.org`
   on a free port, in-process;
4. the hub registration controllers with `SPIFFERegistration` on and the delete guard set to
   `--spiffe-max-svid-lifetime=10s --spiffe-max-clock-skew=1s`, so deletion takes seconds.

Keep the handles in package variables (`spiffeCA`, `spiffeHubAddr`).

## Task 2: the test file

**New file:** `test/integration/registration/spokecluster_spiffe_test.go`

```go
var _ = ginkgo.Describe("Joining Process for spiffe flow", func() {
	var clusterName, workID string
	var agentAPI *fakespire.Server
	var stop context.CancelFunc

	ginkgo.BeforeEach(func() {
		clusterName = fmt.Sprintf("spiffe-%s", rand.String(5))
		workID = fmt.Sprintf("spiffe://ocm-hub.example.org/ocm/cluster/%s/agent/registration", clusterName)
		agentAPI = fakespire.Start(ginkgo.GinkgoT(), spiffeCA)
		agentAPI.SetSVIDs(spiffeCA.Issue(ginkgo.GinkgoT(), workID, 5*time.Second))
		stop = runRegistrationAgentSPIFFE(clusterName, agentAPI.Addr) // spoke agent with --registration-auth=spiffe
	})
	ginkgo.AfterEach(func() { stop() })

	ginkgo.It("registers, is accepted, survives rotation, and is cut off on deny", func() {
		// 1. Register: the ManagedCluster appears, not yet accepted.
		gomega.Eventually(func() error {
			_, err := clusterClient.ClusterV1().ManagedClusters().Get(ctx, clusterName, metav1.GetOptions{})
			return err
		}, eventuallyTimeout, eventuallyInterval).Should(gomega.Succeed())

		// 2. Accept, as clusteradm accept does (no CSR to approve).
		acceptCluster(clusterName)
		gomega.Eventually(joinedCondition(clusterName)).Should(gomega.BeTrue())

		// 3. Rotate: new SVID, old one expires after 5s; the agent keeps updating its lease.
		agentAPI.SetSVIDs(spiffeCA.Issue(ginkgo.GinkgoT(), workID, time.Hour))
		time.Sleep(6 * time.Second)
		gomega.Eventually(leaseRenewedAfter(clusterName, time.Now())).Should(gomega.BeTrue())

		// 4. Deny: the agent's lease stops being renewed.
		denyCluster(clusterName) // hubAcceptsClient: false
		gomega.Consistently(leaseRenewedAfter(clusterName, time.Now()), 5*time.Second).Should(gomega.BeFalse())
	})

	ginkgo.It("can't recreate a deleted cluster until the guard releases it", func() { /* Task 3 */ })
	ginkgo.It("creates no Secret or ConfigMap with key material", func() { /* Task 4 */ })
})
```

`runRegistrationAgentSPIFFE` doesn't exist until Day 16. Today, write the hub side and use a
minimal fake agent instead: a gRPC CloudEvents client built with `NewSPIFFEGRPCOptions` that
publishes a ManagedCluster create and lease updates. Replace it with the real agent on Day 16.

## Task 3: delete guard case

| Step | Input | Expected |
| --- | --- | --- |
| 1 | delete the accepted cluster | object stays, `deletionTimestamp` set, finalizer `open-cluster-management.io/spiffe-revocation` present |
| 2 | the agent tries to create the ManagedCluster again | refused; the object is still terminating |
| 3 | after 11s (10s lifetime + 1s skew) | object is gone |
| 4 | agent with a **new** SVID creates it again | succeeds (that's allowed: SPIRE still issues it identities) |

## Task 4: no key material

```go
ginkgo.It("creates no Secret or ConfigMap with key material", func() {
	for _, ns := range []string{testHubNamespace, clusterName} {
		secrets, _ := kubeClient.CoreV1().Secrets(ns).List(ctx, metav1.ListOptions{})
		for _, s := range secrets.Items {
			for k, v := range s.Data {
				gomega.Expect(looksLikeKeyOrCert(v)).To(gomega.BeFalse(), "%s/%s key %s", ns, s.Name, k)
			}
		}
		// same for ConfigMaps
	}
})

func looksLikeKeyOrCert(b []byte) bool {
	return bytes.Contains(b, []byte("PRIVATE KEY")) || bytes.Contains(b, []byte("BEGIN CERTIFICATE"))
}
```

Exclude ConfigMaps that are clearly not OCM's (for example `kube-root-ca.crt`), and list them in a
comment.

## Task 5: run

```shell
dgo 'make test-registration-integration ARGS="-ginkgo.focus=spiffe"' 2>&1 | tail -4
dgo 'make test-registration-integration' 2>&1 | tail -3        # the whole suite still passes
```

**Output:**

```text
Ran 3 of 105 Specs in 48.2 seconds
SUCCESS! -- 3 Passed | 0 Failed | 0 Pending | 102 Skipped
...
Ran 105 of 105 Specs in 1080.5 seconds
SUCCESS! -- 105 Passed | 0 Failed | 0 Pending | 0 Skipped
```

## Task 6: commit

```shell
git add -A && git commit -s -m ":white_check_mark: integration test for the SPIFFE hub flow"
```

## Rules check

- [ ] The test proves no Secret or ConfigMap with key material appears on the hub.
- [ ] The fake agent has nothing but an SVID.

## Update: agent ID templates

Run the registration flow twice: once with the default templates, and once with a second gRPC
server (another free port) started with
`--spiffe-agent-id-template=spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}`. On
that server, the fake agent with `spiffe://ocm-hub.example.org/k8s/cluster9/ns/team-a-ocm/ocm/registration`
registers `cluster9`, and an agent with the default layout is refused. One flow (register, accept,
lease update) is enough; the rest of the suite only uses the defaults.
