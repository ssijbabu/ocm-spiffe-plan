# Day 13: the delete guard

**Goal:** a deleted cluster can't be brought back by its agent while an SVID issued before the
deletion is still valid.

**Repo:** `ocm`, branch `spire`, package `pkg/registration/hub/managedcluster` (or a new
controller next to it).

## Before you start

| Input | Where |
| --- | --- |
| ManagedCluster controller | `pkg/registration/hub/managedcluster/controller.go` (adds and removes `ManagedClusterFinalizer`) |
| Settings | ClusterManager `spiffe.maxSVIDLifetime` (default 24h), `spiffe.maxClockSkew` (default 5m), passed to the hub controller as flags |
| Design | proposal, "Deleting a cluster" |

The rule:

```text
release time = deletionTimestamp + maxSVIDLifetime + maxClockSkew
```

Example from the proposal: SVIDs live 1h, `maxSVIDLifetime: 1h`, `maxClockSkew: 5m`, `c1` deleted
at 10:40. The finalizer stays until 11:45. Until then a create fails because the name exists, and
the gate from Day 12 refuses every request from c1's identities.

## Task 1: hub flags

The hub registration controller needs the two durations. Add flags to the hub options (find where
`pkg/registration/hub` gets its options, for example `--enabled-registration-drivers`):

```go
fs.DurationVar(&o.SPIFFEMaxSVIDLifetime, "spiffe-max-svid-lifetime", 24*time.Hour, "...")
fs.DurationVar(&o.SPIFFEMaxClockSkew, "spiffe-max-clock-skew", 5*time.Minute, "...")
```

The cluster-manager operator passes them from the ClusterManager on Day 14.

## Task 2: the finalizer controller

**New file:** `pkg/registration/hub/spiffe/deleteguard.go`

```go
const RevocationFinalizer = "open-cluster-management.io/spiffe-revocation"

type deleteGuard struct {
	patcher  patcher.Patcher[*clusterv1.ManagedCluster, clusterv1.ManagedClusterSpec, clusterv1.ManagedClusterStatus]
	lister   clusterlisterv1.ManagedClusterLister
	lifetime time.Duration
	skew     time.Duration
	now      func() time.Time
}

func (g *deleteGuard) sync(ctx context.Context, syncCtx factory.SyncContext, name string) error {
	c, err := g.lister.Get(name)
	if apierrors.IsNotFound(err) {
		return nil
	}
	if err != nil {
		return err
	}
	if c.DeletionTimestamp.IsZero() {
		_, err := g.patcher.AddFinalizer(ctx, c, RevocationFinalizer)
		return err
	}
	release := c.DeletionTimestamp.Add(g.lifetime + g.skew)
	if wait := release.Sub(g.now()); wait > 0 {
		syncCtx.Queue().AddAfter(name, wait) // come back at release time; no polling
		return nil
	}
	return g.patcher.RemoveFinalizer(ctx, c, RevocationFinalizer)
}
```

Only clusters registered through `spiffe` need it. Decide how to tell: for example a label or
annotation the gRPC cluster service sets when a SPIFFE agent creates the cluster, or "all clusters
when `spiffe` is the only driver". Write the choice in the code comment and in
`plan/questions.md`.

Don't let this finalizer block the other cleanup: the existing controller removes the cluster's
RBAC and namespace content as today. Only the object stays, so the name can't be reused.

## Task 3: tests

**File:** `pkg/registration/hub/spiffe/deleteguard_test.go`, with a fake clock and the fake
cluster client.

| Case | Input | Expected output |
| --- | --- | --- |
| Adds finalizer | new cluster `c1` | patch adds `open-cluster-management.io/spiffe-revocation` |
| Waits | deleted at 10:40; now 10:41; lifetime 1h, skew 5m | no patch; requeued after 64m |
| Releases | same; now 11:45:01 | patch removes the finalizer |
| Exact boundary | now exactly 11:45:00 | releases (`wait` is 0) |
| Default 24h | deleted 10:40, defaults | requeued until 10:45 the next day |
| SPIRE clock ahead | SVID issued at 10:39:30 by a clock 4m ahead, so `NotAfter` = 11:43:30 hub time | 11:45 release is still later, so the SVID has expired first |
| Recreate blocked | try to create `c1` while terminating | `AlreadyExists` |

Output check for the "waits" case:

```go
require.Empty(t, clusterClient.Actions())             // no patch
require.Equal(t, 64*time.Minute, queue.lastAddAfter)  // from a fake queue
```

**Run:**

```shell
dgo 'go test ./pkg/registration/hub/spiffe/... -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

**Output:**

```text
--- PASS: TestDeleteGuardAddsFinalizer (0.00s)
--- PASS: TestDeleteGuardWaits (0.00s)
--- PASS: TestDeleteGuardReleases (0.00s)
--- PASS: TestDeleteGuardBoundary (0.00s)
--- PASS: TestDeleteGuardDefault (0.00s)
ok  	open-cluster-management.io/ocm/pkg/registration/hub/spiffe	0.05s
```

## Task 4: wire it into the hub manager

Start the controller in `pkg/registration/hub/manager.go` when `SPIFFERegistration` is on and
`spiffe` is among the enabled drivers. Check the existing gate pattern there (for example how the
`grpc` driver's controllers are started).

## Task 5: commit

```shell
git add -A && git commit -s -m ":sparkles: hold deleted SPIFFE clusters until every SVID has expired

Add the open-cluster-management.io/spiffe-revocation finalizer and keep it
until deletionTimestamp + maxSVIDLifetime + maxClockSkew."
```

## Rules check

- [ ] OCM doesn't contact SPIRE; it only waits for a known upper bound.
- [ ] The procedure note stays in the docs: remove the cluster's `ClusterSPIFFEID`s before
      deleting the `ManagedCluster`.
