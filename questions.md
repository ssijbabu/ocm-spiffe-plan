# Questions

Open points found while implementing. Tick them when answered, and say where.

- [ ] Q1 (proposal OQ4): which SPIRE topology for e2e? Decide by Day 26. Note: with one hub
      server and agent-only spokes, spoke `ClusterSPIFFEID`s aren't reconciled (Day 26 update).
- [ ] Q2: does envtest in ocm run Kubernetes >= 1.30? Needed for ValidatingAdmissionPolicy on Day 19.
- [ ] Q3: exact identity of the work agent on the managed cluster in hosted mode, for the add-on
      admission policy (Day 20).
- [x] Q4 (Day 2): can the ClusterManager CRD check that `serverSPIFFEID` is in `trustDomain`
      with CEL? **No.** `registrationDrivers` has no `maxItems`, so any string rule on its items
      exceeds the CRD validation cost budget (5x). Adding `maxItems` would change an existing
      upstream field. Decision: the pattern checks it is a SPIFFE ID; the cluster-manager operator
      checks the trust domain and sets `Degraded` (added to Day 14).
- [x] Q5 (Day 3): does the same cost limit hit CEL rules on the Klusterlet? **No.** `registrationConfiguration`
      is not a list there; the CRD installs and the rules work (api `a770a061`).

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

- [x] Q7 (2026-11-13, proposal OQ1): should the SPIFFE ID layout be configurable? **Yes, with
      agent ID templates.** ClusterManager and Klusterlet get `agentIDTemplate` (default
      `spiffe://{trustDomain}/ocm/cluster/{cluster}/agent/{role}`) and `addOnAgentIDTemplate`
      (default `spiffe://{trustDomain}/ocm/cluster/{cluster}/addon/{addon}/agent/{agent}`).
      Placeholders fill whole path segments; `{cluster}`, `{role}` / `{addon}`, `{agent}` are
      required; `{namespace}` is optional and never used for authorization. The hub parses with
      its templates (sdk-go `idtemplate`, `IDScheme`, Day 4); the Klusterlet operator renders each
      agent's full ID into `--spiffe-agent-id` (Day 18), so `clusterName` becomes required with
      `spiffe`. Both sides must use the same templates; the hub accepts one pair at a time, so
      changing it on a live hub interrupts clusters until their Klusterlets match. Rejected:
      free-form IDs passed to agents (the hub would need a per-cluster ID binding, trusted on first
      connect). Plan: Day 4 rewritten; "Update: agent ID templates" in Days 2, 3, 5, 8, 10, 11, 14–18, 22–24, 26–28 and
      30, and `clusterspiffeid.md`. The proposal on `spire` doesn't have this yet (Day 30).
- [ ] Q8 (2026-11-13): how does an add-on agent learn `hubEndpoint`, `hubSPIFFEID`, its trust
      domain and `addOnAgentIDTemplate`? None is secret. Day 23 left the first two open ("decide in
      the PR"); the template has the same answer. Decide by Day 23.
- [x] Q9 (Day 9, 2026-11-13): how does the CloudEvents gRPC client reconnect? The base client
      (`generic/clients/baseclient.go`) reconnects by itself when the transport reports an error:
      the subscribe stream fails, or the connection goes `Idle` or `TransientFailure` (the v2
      transport's `monitorConnectionState`). It closes the transport, waits `DelayFn`
      (`wait.Backoff{Duration: 5s, Factor: 5, Jitter: 1.0, Cap: 1m}`, reset after 10 minutes
      without backoff, so the first retry is after 5–10s), connects (a new `grpc.ClientConn`, so
      a new TLS handshake with the current SVID), subscribes again and resyncs. No change needed
      in sdk-go. An agent therefore misses events for 5–10s after each SVID expiry; with a 1h TTL
      that's once an hour per agent.
- [ ] Q10 (Day 7/8, 2026-11-13): the plan's go-spiffe v2.6.0 has a data race in
      `X509Source.GetX509BundleForTrustDomain`; sdk-go uses v2.9.0, which moves sdk-go's grpc from
      1.76.0 to 1.79.3. Mention it in the sdk-go PR.

## Lessons from implementation (2026-11-13)

- **gofmt rewrites quotes in doc comments.** `''` in a `+kubebuilder:validation:XValidation`
  marker becomes `”`, which breaks the generated CEL rule. Avoid empty-string literals in markers
  (use `size(x) > 0`), and grep the CRDs after `make update`.
- **The SPIRE Helm chart ignores `ClusterSPIFFEID`s without `spec.className`**
  (`watchClassless: false`; class `<namespace>-<release>`, `spire-server-spire` on ocm-hub).
- **Test TLS through a real handshake.** go-spiffe verifies peers in `VerifyPeerCertificate`, so
  `VerifiedChains` is empty; a unit test that builds a TLS context by hand hid that the planned
  authenticator refused every agent.
- **Run SPIFFE tests with `-race`.** It caught go-spiffe v2.6.0's bundle race.
- **api has Ginkgo v1 without `extensions/table`.** Write table cases as loops of `It`.
- **`grpcoptions.NewGRPCOptions()` leaves `Dialer` nil.** Build the `GRPCDialer` yourself.
- **CloudEvents test events need the `contenttype` attribute and data**, or the receiver drops them
  as `unknown Message encoding`.

