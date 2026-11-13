# Day 25: migration from the `grpc` driver, and downgrade

**Goal:** a hub can run `grpc` and `spiffe` side by side; one cluster moves from `grpc` to `spiffe`
without losing anything; and it can move back.

**Repo:** `ocm`, branch `spire`, `test/integration/registration` and the Klusterlet operator.

## Before you start

| Input | Where |
| --- | --- |
| Both listeners | Day 10 |
| Design | proposal, Upgrade / Downgrade Strategy |

What must stay the same across the switch: the ManagedCluster, its namespace, its ManifestWorks,
its add-ons and their RBAC. The OCM user and group names are the same for `grpc` and `spiffe`, so
hub RBAC doesn't change.

## Task 1: the switch in the operator

When a Klusterlet changes from `grpc` (or `csr`) to `spiffe`:

1. Run first activation (Day 21): agents stop, namespaces are cleaned, the policy is recorded.
2. Start the agents with SVIDs.
3. Once the registration agent reports the cluster available again, delete the old
   `hub-kubeconfig-secret`, `bootstrap-hub-kubeconfig` and the add-on `<addon>-hub-kubeconfig`
   Secrets.

When it changes back (downgrade): remove the admission policies and binding, clear
`status.spiffe`, and wait for a bootstrap Secret as for a new join.

## Task 2: integration test, side by side

**New spec** in `test/integration/registration/spokecluster_spiffe_test.go`:

| Step | Input | Expected |
| --- | --- | --- |
| 1 | hub with `grpc` and `spiffe` drivers; cluster `a` joins with `grpc`, cluster `b` with `spiffe` | both available |
| 2 | ManifestWork for each | both applied, status synced |
| 3 | `a` lists ManifestWorks of `b` | refused (as today) |

## Task 3: integration test, migrating one cluster

| Step | Input | Expected |
| --- | --- | --- |
| 1 | cluster `m` joined with `grpc`, accepted, with ManifestWork `w1` and add-on `my-addon` | available |
| 2 | record the ManagedCluster UID, the `w1` UID, the MCA UID and the cluster's hub RoleBindings | |
| 3 | switch `m`'s agents to `spiffe` (new SVIDs from the fake Workload API) | available again within the timeout |
| 4 | compare | same ManagedCluster UID, same `w1` UID and status, same MCA, same RoleBindings |
| 5 | Secrets on the spoke | `hub-kubeconfig-secret` and `my-addon-hub-kubeconfig` gone |
| 6 | hub | the cluster's CSRs are not recreated |

Example comparison:

```go
gomega.Expect(after.UID).To(gomega.Equal(before.UID), "ManagedCluster must not be recreated")
gomega.Expect(workAfter.UID).To(gomega.Equal(workBefore.UID))
gomega.Expect(meta.IsStatusConditionTrue(workAfter.Status.Conditions, "Applied")).To(gomega.BeTrue())
```

## Task 4: downgrade test

| Step | Input | Expected |
| --- | --- | --- |
| 1 | from the end of Task 3, switch `m` back to `grpc` with a new bootstrap kubeconfig | available |
| 2 | admission policies on the spoke | removed |
| 3 | ManagedCluster UID | unchanged |

## Task 5: removing `grpc` from the hub

| Step | Input | Expected |
| --- | --- | --- |
| 1 | hub with both drivers, all clusters on `spiffe` | |
| 2 | remove `grpc` from `registrationDrivers` | the `grpc` listener stops; `cluster-manager-grpc-server` Service and the gRPC serving cert, signer and bootstrap token Secrets are deleted |
| 3 | `spiffe` clusters | still available |

**Run:**

```shell
dgo 'make test-registration-integration ARGS="-ginkgo.focus=spiffe"' 2>&1 | tail -3
dgo 'make test-registration-operator-integration' 2>&1 | tail -2
```

## Task 6: commit

```shell
git add -A && git commit -s -m ":white_check_mark: migration between grpc and spiffe drivers"
git push origin spire
```

## Rules check

- [ ] After migration, no Secret with hub credentials is left on the spoke.

## Status (2026-11-13)

Done, committed locally. See "Progress" in `questions.md` and Q14-Q17.

## In simple words: what changed and why

Day 25 was about clusters that already exist. A hub can run the old drivers and `spiffe` side by
side, and a cluster can move from one to the other and back. What makes this possible is that the
OCM user and group names are the same in every driver, so the cluster's permissions on the hub
don't change when it switches.

When a cluster switches to `spiffe`, its old Secrets with hub credentials have to go, or the
cluster would still hold long-lived credentials. The registration agent deletes them, and only
after its new SVID-based connection shows the cluster as accepted. The agent does this, not the
operator, because only the agent knows the switch really worked. Until that moment the old
Secrets are still there, so an admin can switch back if something is wrong. The old add-on Secret
is deleted when the new add-on ConfigMap is written.

When a cluster leaves `spiffe`, the operator undoes what it set up: the admission policies, the
add-on namespace labels and the `spiffe` part of the status. The one policy that is shared by all
Klusterlets is removed only when no other Klusterlet still uses `spiffe`.

The integration test moves a cluster from `csr` to `spiffe` and back. The `ManagedCluster`, its
`ManifestWork`, its add-on and its permissions stay the same objects the whole time, and no
certificate request is made while the cluster is on `spiffe`. The plan said to test from `grpc`;
we used `csr` because the `grpc` test hub allows everything and couldn't show a refusal
(`questions.md` Q17). A unit test also checks that removing `grpc` from a hub removes the old
Service and keeps the server running for the SPIFFE listener.
