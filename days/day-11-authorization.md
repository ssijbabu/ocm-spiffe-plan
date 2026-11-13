# Day 11: authorize every request by the identity in the SVID

**Goal:** on the SPIFFE listener, an agent can only act on the cluster in its SVID, an add-on agent
only on its own add-on, and CSR and token requests are refused.

**Repos:** `sdk-go` (authorizer) and `ocm` (vendor update and tests).

## Before you start

| Input | Where |
| --- | --- |
| SAR authorizer | sdk-go `pkg/cloudevents/server/grpc/authz/kube/sar.go` |
| Existing cluster check | sdk-go `pkg/server/grpc/authn/clusteridentity.go`: `ValidateClusterIdentity` |
| SPIFFE identity in context | Day 5, `OCMIdentityFromContext` |

What happens today in `SARAuthorizer.authorize`:

```go
if eventsType.CloudEventsDataType == csr.CSREventDataType {
	if err := authn.ValidateClusterIdentity(ctx, cluster); err != nil {
		return authz.DecisionDeny, err
	}
}
// then a SubjectAccessReview with the user and groups from the context
```

So the cluster in the identity is checked only for CSR events. For other events it relies on
`ce-clustername` matching the object (`validateEventClusterName`) and on RBAC. With SPIFFE, the
identity comes from SPIRE, so check it on every event.

## Task 1: SPIFFE checks in the authorizer (`sdk-go`)

**Change** `sar.go`:

```go
func (s *SARAuthorizer) authorize(ctx context.Context, cluster string, eventsType types.CloudEventsType, metaObj metav1.ObjectMeta) (authz.Decision, error) {
	if id, ok := authn.OCMIdentityFromContext(ctx); ok {
		if err := authorizeSPIFFE(id, cluster, eventsType, metaObj); err != nil {
			return authz.DecisionDeny, err
		}
	} else if eventsType.CloudEventsDataType == csr.CSREventDataType {
		if err := authn.ValidateClusterIdentity(ctx, cluster); err != nil {
			return authz.DecisionDeny, err
		}
	}
	// ... SubjectAccessReview as today ...
}

// authorizeSPIFFE applies the checks that come from the SVID itself.
func authorizeSPIFFE(id authn.OCMIdentity, cluster string, t types.CloudEventsType, meta metav1.ObjectMeta) error {
	if id.ClusterName != cluster {
		return fmt.Errorf("%s may not act on cluster %q", id.User, cluster)
	}
	switch t.CloudEventsDataType {
	case csr.CSREventDataType, serviceaccount.TokenRequestDataType:
		return fmt.Errorf("%s is not served on the SPIFFE listener", t.CloudEventsDataType)
	}
	if id.IsAddOn() {
		switch t.CloudEventsDataType {
		case v1alpha1.ManagedClusterAddOnEventDataType, v1beta1.ManagedClusterAddOnEventDataType:
			if meta.Name != "" && meta.Name != id.AddOnName {
				return fmt.Errorf("%s may not act on add-on %q", id.User, meta.Name)
			}
		}
	}
	return nil
}
```

Do the same in `AuthorizeStream` (subscribe requests carry the cluster name too; check it against
`id.ClusterName`).

Look at what an add-on agent really sends (lease updates, events, add-on status) and list which
event types it may use. Anything else falls to the SubjectAccessReview, which uses the add-on's
own RBAC.

## Task 2: tests (`sdk-go`)

**File:** `pkg/cloudevents/server/grpc/authz/kube/sar_test.go`. Put an `OCMIdentity` in the
context the way the authenticator does, and use the fake kube client for the SAR (allow all, so
only the SPIFFE checks decide).

| Identity (from SVID) | Event | Expected |
| --- | --- | --- |
| cluster1 work agent | ManifestWork status update, `ce-clustername: cluster1` | Allow |
| cluster2 work agent | ManifestWork status update, `ce-clustername: cluster1` | Deny: `may not act on cluster "cluster1"` |
| cluster1 registration agent | ManagedCluster status `cluster1` | Allow |
| cluster1 registration agent | CSR create | Deny: `not served on the SPIFFE listener` |
| cluster1 `my-addon` agent | ManagedClusterAddOn `my-addon` in `cluster1` | Allow |
| cluster1 `my-addon` agent | ManagedClusterAddOn `other-addon` in `cluster1` | Deny: `may not act on add-on "other-addon"` |
| cluster1 `my-addon` agent | ManagedClusterAddOn `my-addon` in `cluster2` | Deny: `may not act on cluster "cluster2"` |
| cluster1 `my-addon` agent | token request | Deny |
| no SPIFFE identity (token user) | CSR create, matching cluster | unchanged behaviour (existing tests) |

Also check the SubjectAccessReview the fake client receives: user and groups are exactly the
mapped ones, including `system:authenticated`.

```go
sar := fakeKube.Actions()[0].(clienttesting.CreateAction).GetObject().(*authorizationv1.SubjectAccessReview)
require.Equal(t, "system:open-cluster-management:cluster1:work", sar.Spec.User)
require.ElementsMatch(t, []string{
	"system:open-cluster-management:cluster1",
	"system:open-cluster-management:managed-clusters",
	"system:authenticated",
}, sar.Spec.Groups)
```

**Run:**

```shell
cd /home/ssijbabu/Documents/repos/ocm/sdk-go
dgo 'go test ./pkg/cloudevents/server/grpc/authz/... -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

**Output:** all cases pass, existing tests unchanged.

## Task 3: move ocm to the new sdk-go

```shell
git -C ../sdk-go commit -s -am ":sparkles: check the SPIFFE identity on every gRPC event" && git -C ../sdk-go push fork spire
cd ../ocm
dgo 'go mod edit -replace open-cluster-management.io/sdk-go=github.com/ssijbabu/sdk-go@'"$(git -C ../sdk-go rev-parse HEAD)"' && go mod tidy && go mod vendor && go build ./...'
git commit -s -am ":seedling: bump sdk-go fork"
```

## Task 4: a hub-level test in ocm

**File:** `pkg/server/grpc/spiffe_listener_test.go`. Start the hub with the SPIFFE listener (Day 10)
and envtest. Create `ManagedCluster` `cluster1` and `cluster2`, accepted, with the usual hub RBAC.

| Client SVID | Action | Expected |
| --- | --- | --- |
| `.../cluster/cluster2/agent/work` | publish a status update for a ManifestWork in namespace `cluster1` | `PermissionDenied`; the ManifestWork is unchanged |
| `.../cluster/cluster1/agent/work` | same | accepted; the status is written |

**Run:**

```shell
dgo 'go test ./pkg/server/grpc/... -run CrossCluster -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

## End of day

| Output | Check |
| --- | --- |
| sdk-go: SPIFFE checks in the authorizer | unit tests |
| ocm: cross-cluster test | passes |

## Rules check

- [ ] An agent can never act for another cluster or add-on, even with a valid SVID.
- [ ] No CSR or token path exists on the SPIFFE listener.

## Update: multiple agent trust domains

Add trust domain pinning (by-mode proposal, "Trust domain pinning"):

1. The gRPC cluster service (`pkg/server/services/cluster`) sets `open-cluster-management.io/spiffe-trust-domain: <SVID's trust domain>` when an agent creates its ManagedCluster, replacing any value in the request, and keeps it on agent updates (same pattern as the cluster-set label in `cluster.go`).
2. Every SPIFFE request is refused unless the SVID's trust domain equals the cluster's annotation.
3. No annotation: if `agentTrustDomains` has one entry, treat the cluster as pinned to it; with several, refuse until an admin sets it.

Test rows: cluster2 pinned to `east.example.org`; an SVID `spiffe://west.example.org/ocm/cluster/cluster2/agent/work` is refused, and so is an agent trying to change the annotation.

## Update: agent ID templates

Nothing changes in the authorizer: it uses `OCMIdentity.ClusterName`, `Role`, `AddOnName` and
`TrustDomain`, which `IDScheme.Parse` fills the same way for any template. Add one test row with
the namespace template from Day 4 to show `Namespace` has no effect: cluster1's work agent with
`{namespace}` = `team-a-ocm` may update cluster1's ManifestWork status, and the same agent may
not touch cluster2's.

**Rule:** never authorize on `OCMIdentity.Namespace`. The hub can't verify it.

## Status (2026-11-13)

Done, committed locally. See "Progress (Days 10-23)" in `questions.md` for the commit and findings.
