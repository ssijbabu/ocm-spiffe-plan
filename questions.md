# Questions

Open points found while implementing. Tick them when answered, and say where.

- [ ] Q1 (proposal OQ4): which SPIRE topology for e2e? Decide by Day 26.
- [ ] Q2: does envtest in ocm run Kubernetes >= 1.30? Needed for ValidatingAdmissionPolicy on Day 19.
- [ ] Q3: exact identity of the work agent on the managed cluster in hosted mode, for the add-on
      admission policy (Day 20).
- [x] Q4 (Day 2): can the ClusterManager CRD check that `serverSPIFFEID` is in `trustDomain`
      with CEL? **No.** `registrationDrivers` has no `maxItems`, so any string rule on its items
      exceeds the CRD validation cost budget (5x). Adding `maxItems` would change an existing
      upstream field. Decision: the pattern checks it is a SPIFFE ID; the cluster-manager operator
      checks the trust domain and sets `Degraded` (added to Day 14).
- [x] Q5 (Day 3): same cost limit may hit CEL rules on the Klusterlet. **No**: `registrationConfiguration`
      is not a list there, so it should be fine; confirm when the CRD installs.

## Baseline (Day 1, 2026-11-11)

| Repo | Command | Result |
| --- | --- | --- |
| api | `make test-api-integration` | 174 of 174 specs pass |
| api | `go test ./...` | only `test/integration/api` fails (needs envtest; expected) |
| sdk-go | `make test-unit` | 52 packages ok, 0 fail |
| ocm | `make test-unit` | 108 packages ok, 0 fail |
| clusteradm | `make test` | pass |
| addon-framework | `make test-unit` | 11 packages ok, 0 fail |

Heads at baseline: api `08ef6f77`, sdk-go `fb8d1d0f`, ocm `23f2c4deb`, clusteradm `f02a8843`,
addon-framework `main` at clone time.

- [ ] Q6 (2026-11-11): multiple agent trust domains were in the `spire-impl-by-mode` version of
      the proposal but not in `spire` (PR #203), so Days 2-3 first missed `agentTrustDomains`.
      Added to the API in api `c77b5174`, and to the affected day documents ("Update: multiple
      agent trust domains"). The proposal on `spire` still describes one trust domain.
