# SPIFFE e2e results (2026-11-13)

Environment: `ocm/test/e2e/spiffe/` on kind (Kubernetes 1.37), SPIRE chart 0.30.4 (SPIRE 1.15.5),
one SPIRE server on the hub (`ocm-e2e-hub`), agents on `ocm-e2e-spoke`; hosted modes use
`ocm-e2e-spoke` as the management cluster and `ocm-e2e-managed`, `ocm-e2e-managed2` as managed
clusters (no SPIRE there: the agents run on the management cluster). OCM images built from
`spire` (ocm `acdb176aa`..`301d209f2`), clusteradm `1c9b4a71`. Hub has the spiffe driver only.

| Spec | Default (cluster1) | Singleton (cluster2) | Hosted (cluster4) | SingletonHosted (cluster5) |
| --- | --- | --- | --- | --- |
| join with no token, entries after SPIFFEAdmissionReady, accept | pass | pass | pass | pass |
| no hub credential Secret on the agents' cluster | pass | pass | pass (Q15: external-managed-kubeconfig*) | pass |
| cluster pinned to the trust domain | pass | pass | pass | pass |
| 1 Work: ManifestWork Applied + Available, ConfigMap on the right cluster | pass | pass | pass (on managed, not management) | pass |
| 2 Status feedback | not run | not run | not run | not run |
| 3 Add-on enable | not run (needs an add-on image built on `spiffeclient`) | | | |
| 4 Rotation under load (TTL 2m, 13 updates over 5m) | nothing lost; 1 update late (Q18) | | | |
| 5 Agent down past expiry | covered by 4 (agents reconnect with rotated SVIDs) | | | |
| 6 Add-on revoke | not run (needs 3) | | | |
| 7 Cluster deny | pass after fix `2bdd76cff` | | | |
| 8 Cluster delete: guard holds for maxSVIDLifetime + maxClockSkew | pass (3m + 1m; released on time) | | | |
| 9 exec into an agent refused | pass | pass | pass | |
| 11 tampered policy: recovery for that Klusterlet only | | | pass after fix `301d209f2` (6s; cluster5 untouched) | |
| Klusterlet delete removes its policies | pass after fix `8d1a4d175` | pass | | |
| 10, 12 (two Klusterlets on one managed cluster) | not run: each hosted Klusterlet here has its own managed cluster | | | |
| Custom agent ID template (Day 27, Task 7) | not run | | | |

## Bugs the e2e found (all fixed, with unit tests)

| Commit | Bug |
| --- | --- |
| ocm `9b2cd94bd` | Klusterlet status write conflicts lost the recovery record: agents started and were deleted again forever; the add-on namespace's record was lost and the namespace unlabeled. Recovery now ends the pass after recording; SPIFFE status fields are re-written on conflict. |
| ocm `9b2cd94bd` | The work agent's spiffe-svid health check failed before leader election, so the liveness probe killed it before it could start. |
| ocm `2bdd76cff` | A denied cluster's work agent reconnected at once (the hub keeps the work RoleBinding on deny). The gate refuses work and add-on agents of a cluster that isn't accepted. |
| ocm `8d1a4d175` | Deleting a spiffe Klusterlet left its admission policies. |
| ocm `301d209f2` | A deleted agent policy went unnoticed until the 5-minute resync. The operator now watches policies and bindings. |
| clusteradm `1c9b4a71` | `init` told admins to create SPIRE entries before `join`; they come after SPIFFEAdmissionReady. |

## Notes

- Snap `kubectl` prints nothing unless its output is piped; the scripts pipe through `cat`.
- `kubectl rollout restart` of agent Deployments is refused by the agent policy (only the operator
  may change them). Restart agents by deleting their pods.
- A cluster deleted after its Klusterlet keeps its ManifestWorks' finalizers (upstream behavior);
  the spiffe-revocation finalizer itself was released on time.
