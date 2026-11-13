# Day 12: revocation, closing connections when a cluster or add-on is cut off

**Goal:** when an admin denies or deletes a cluster, or disables an add-on, the hub closes the
matching gRPC connections right away and refuses new ones.

**Repo:** `ocm`, branch `spire`, package `pkg/server/grpc`.

## Before you start

| Input | Where |
| --- | --- |
| Connection registry | Day 8, `Registry.CloseCluster`, `Registry.CloseAddOn` |
| Informers in the server | `pkg/server/grpc/clients.go` (`ClusterInformers`, `AddOnInformers`) |
| Design | proposal, "Revoke" and the gRPC server watch paragraph |

| Admin action on the hub | What the server does |
| --- | --- |
| `hubAcceptsClient: false` on `cluster1` | close all of cluster1's connections (Klusterlet and add-on agents) |
| delete `ManagedCluster/cluster1` | same, as soon as `deletionTimestamp` is set |
| delete `ManagedClusterAddOn/my-addon` in namespace `cluster1` | close only `my-addon`'s agents on cluster1 |

## Task 1: the revocation controller

**New file:** `pkg/server/grpc/revocation.go`

```go
// revocationHandlers closes connections when access is withdrawn.
func revocationHandlers(reg *spiffeserver.Registry) (clusterH, addonH cache.ResourceEventHandler) {
	closeCluster := func(c *clusterv1.ManagedCluster, why string) {
		if n := reg.CloseCluster(c.Name); n > 0 {
			klog.Infof("closed %d connection(s) of cluster %s: %s", n, c.Name, why)
		}
	}
	clusterH = cache.ResourceEventHandlerFuncs{
		UpdateFunc: func(_, obj any) {
			c := obj.(*clusterv1.ManagedCluster)
			switch {
			case !c.DeletionTimestamp.IsZero():
				closeCluster(c, "cluster is being deleted")
			case !c.Spec.HubAcceptsClient:
				closeCluster(c, "hubAcceptsClient is false")
			}
		},
		DeleteFunc: func(obj any) {
			if c, ok := tombstone[*clusterv1.ManagedCluster](obj); ok {
				closeCluster(c, "cluster deleted")
			}
		},
	}
	addonH = cache.ResourceEventHandlerFuncs{
		UpdateFunc: func(_, obj any) {
			a := obj.(*addonv1beta1.ManagedClusterAddOn)
			if !a.DeletionTimestamp.IsZero() {
				reg.CloseAddOn(a.Namespace, a.Name)
			}
		},
		DeleteFunc: func(obj any) {
			if a, ok := tombstone[*addonv1beta1.ManagedClusterAddOn](obj); ok {
				reg.CloseAddOn(a.Namespace, a.Name)
			}
		},
	}
	return
}
```

Register these on the informers in `runSPIFFEListener` (Day 10). `tombstone` unwraps
`cache.DeletedFinalStateUnknown`.

A cluster that has never been accepted can still create its `ManagedCluster` and update its status,
as today with the `grpc` driver, so don't close on `hubAcceptsClient: false` for a cluster that has
just been created. Check the existing pre-acceptance permissions and keep the same behaviour.
Closing is about *changes*: accepted, then denied.

## Task 2: refuse new connections

Closing isn't enough: the agent reconnects within a second. The SubjectAccessReview already
refuses most requests once the hub RBAC for a denied cluster is removed, but don't rely on timing.
Add a check in the SPIFFE path of the authorizer (or a small interceptor in ocm, which has the
cluster lister):

```go
// clusterGate refuses requests from clusters that are terminating or no longer accepted.
func clusterGate(lister clusterlisterv1.ManagedClusterLister) func(ctx context.Context) error {
	return func(ctx context.Context) error {
		id, ok := grpcauthn.OCMIdentityFromContext(ctx)
		if !ok {
			return nil
		}
		c, err := lister.Get(id.ClusterName)
		switch {
		case apierrors.IsNotFound(err):
			return nil // not created yet: the pre-acceptance rules apply
		case err != nil:
			return status.Error(codes.Unavailable, err.Error())
		case !c.DeletionTimestamp.IsZero():
			return status.Errorf(codes.PermissionDenied, "cluster %s is being deleted", c.Name)
		}
		return nil
	}
}
```

`hubAcceptsClient: false` is enforced by the existing RBAC (the hub removes the cluster's
permissions), so the gate only needs the terminating case. Confirm this against
`pkg/registration/hub` before relying on it.

## Task 3: tests

**File:** `pkg/server/grpc/revocation_test.go` (unit, with a fake registry) and an integration
case in `spiffe_listener_test.go`.

| Case | Input | Expected output |
| --- | --- | --- |
| Deny | cluster1 accepted; work agent streaming; set `hubAcceptsClient: false` | stream ends within 1s; log `closed 1 connection(s) of cluster cluster1: hubAcceptsClient is false` |
| Delete | delete `ManagedCluster/cluster1` (finalizer keeps it terminating) | stream ends; reconnect refused with `cluster cluster1 is being deleted` |
| Add-on disable | work agent and `my-addon` agent streaming; delete `ManagedClusterAddOn/my-addon` | only the add-on stream ends |
| Other cluster untouched | cluster2 agent streaming while cluster1 is denied | cluster2 keeps streaming |
| New cluster | cluster3 not yet accepted connects and creates its ManagedCluster | allowed, as with `grpc` today |

**Run:**

```shell
dgo 'go test ./pkg/server/grpc/... -run "Revocation|Deny|Delete|Disable" -race -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

**Output:**

```text
--- PASS: TestRevocationDeny (1.21s)
--- PASS: TestRevocationDelete (1.34s)
--- PASS: TestRevocationAddOnDisable (1.18s)
--- PASS: TestRevocationOtherClusterUntouched (1.25s)
ok  	open-cluster-management.io/ocm/pkg/server/grpc	7.3s
```

## Task 4: commit

```shell
git add -A && git commit -s -m ":sparkles: close SPIFFE connections on cluster deny, delete and add-on disable"
```

## Rules check

- [ ] Revocation needs no SPIRE call: OCM only closes connections and refuses requests.

## Update: multiple agent trust domains

Removing a trust domain from `agentTrustDomains` closes every connection from that domain (`Registry.CloseTrustDomain`) and refuses new ones at TLS.

## Update: what Days 4–9 built (2026-11-13)

The sdk-go registry already has `CloseCluster(cluster)`, `CloseAddOn(cluster, addon)` and
`CloseTrustDomain(td)` (each returns how many connections it closed), plus `Len()` and
`PeerSerials(id)` for tests. A closed agent reconnects at once at the TLS level (Day 9 test); the
authorizer must refuse it, so revocation needs both the close and the authorizer check.
