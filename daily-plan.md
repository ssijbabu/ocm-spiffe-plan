# SPIFFE gRPC registration: daily coding plan

This plan turns the enhancement proposal into code, one working day at a time.

- **Design source:** `enhancements/enhancements/sig-architecture/199-spiffe-grpc-registration/README.md`
  on branch `spire` (PR open-cluster-management-io/enhancements#203). When the plan and the
  proposal disagree, the proposal wins; update this plan.
- **Branches:** work on `spire` in `api`, `sdk-go`, `ocm` and `clusteradm`. Each one starts at
  current upstream `main`. The old `spiffe-registration` and `spiffe-hub-signer` branches are from
  the earlier hub-signer design. Don't merge them; copy small pieces only if they still fit.
- **Not cloned yet:** `addon-framework` (needed from Day 23).
- **Go toolchain:** Go is not installed on this machine. Run Go and make in the local
  `golang:1.27` image, as for the Azure driver: mount the workspace and `~/go/pkg/mod`, run as your
  own user.

## Day documents

Each day has its own document in [days/](days/), with inputs, steps, full examples and expected
output for every task. Run Go through [tools/dgo](tools/dgo).

SPIRE entries for every workload and mode, and how they follow the agent ID templates:
[clusterspiffeid.md](clusterspiffeid.md).

| Day | Document | Repo |
| --- | --- | --- |
| 1 | [Set up](days/day-01-setup.md) | all |
| 2 | [API: ClusterManager and feature gate](days/day-02-api-clustermanager.md) | api |
| 3 | [API: Klusterlet, status, add-on](days/day-03-api-klusterlet-addon.md) | api |
| 4 | [SPIFFE ID templates and mapping](days/day-04-sdkgo-id-mapping.md) | sdk-go |
| 5 | [SPIFFE authenticator](days/day-05-sdkgo-authenticator.md) | sdk-go |
| 6 | [Fake Workload API](days/day-06-sdkgo-fake-workload-api.md) | sdk-go |
| 7 | [SVID picking and client TLS](days/day-07-sdkgo-svid-picker-client.md) | sdk-go |
| 8 | [Server TLS and connection registry](days/day-08-sdkgo-server-tls-registry.md) | sdk-go |
| 9 | [Renewal end to end](days/day-09-sdkgo-renewal.md) | sdk-go |
| 10 | [SPIFFE listener](days/day-10-ocm-spiffe-listener.md) | ocm |
| 11 | [Authorization by SVID](days/day-11-authorization.md) | sdk-go, ocm |
| 12 | [Revocation](days/day-12-revocation.md) | ocm |
| 13 | [Delete guard](days/day-13-delete-guard.md) | ocm |
| 14 | [Hub operator](days/day-14-hub-operator.md) | ocm |
| 15 | [Hub integration test](days/day-15-hub-integration-test.md) | ocm |
| 16 | [Spoke registration driver](days/day-16-spoke-driver.md) | ocm |
| 17 | [Work and singleton agents](days/day-17-work-singleton-agents.md) | ocm |
| 18 | [Klusterlet operator, four modes](days/day-18-klusterlet-operator-modes.md) | ocm |
| 19 | [Admission policy, agent namespaces](days/day-19-admission-policy-agents.md) | ocm |
| 20 | [Admission policy, exec, add-ons, labels](days/day-20-admission-policy-addons-exec.md) | ocm |
| 21 | [Recovery and trusted state](days/day-21-recovery-trusted-state.md) | ocm |
| 22 | [Add-on registration](days/day-22-addon-registration.md) | ocm |
| 23 | [addon-framework](days/day-23-addon-framework.md) | addon-framework |
| 24 | [clusteradm](days/day-24-clusteradm.md) | clusteradm |
| 25 | [Migration from grpc](days/day-25-migration.md) | ocm |
| 26 | [e2e environment](days/day-26-e2e-environment.md) | ocm |
| 27 | [e2e, Default and Singleton](days/day-27-e2e-default-singleton.md) | ocm |
| 28 | [e2e, hosted modes](days/day-28-e2e-hosted.md) | ocm |
| 29 | [AKS check](days/day-29-aks-validation.md) | AKS |
| 30 | [Review and PRs](days/day-30-review-and-prs.md) | all |

## Rules to check every day

These come from `CLAUDE.md`. Check every change against them before committing.

- [ ] Agents use only SPIFFE SVIDs for registration, renewal and revocation.
- [ ] OCM never issues or signs an identity. No CSR, no signer, no token in `spiffe` mode.
- [ ] SVIDs stay in memory. No Secret or ConfigMap holds a key, a certificate or a hub credential.
- [ ] The only outside service OCM talks to is the local SPIRE agent, through the Workload API.
- [ ] It works in all four modes: `Default`, `Singleton`, `Hosted`, `SingletonHosted`.
- [ ] Add-on agents follow the same model, each with its own SVID.
- [ ] Everything new is behind the `SPIFFERegistration` feature gate.

## Order of work

```text
api ──► sdk-go ──► ocm (hub) ──► ocm (agents, operator, admission) ──► addon-framework ──► clusteradm ──► e2e
```

Each repo depends on the one before it. While upstream PRs are open, point the next repo at your
fork commits with a `replace` line, as was done for the Azure driver. Swap the `replace` lines for
released versions before the final PRs merge.

## Week 1: API and the identity core

### Day 1: set up

- [ ] Check out `spire` in every repo and pull upstream `main`.
- [ ] Clone `addon-framework` next to the other repos and create a `spire` branch there.
- [ ] Check the Docker Go wrapper: `go build ./...` passes in all repos.
- [ ] Run the unit tests once on clean `main` and save the results. This is the baseline for
      telling your failures from existing ones.
- [ ] Read the proposal's Implementation Details again and list anything unclear in
      `plan/questions.md`.

**Done when:** every repo builds and the baseline test results are saved.

### Day 2: API, ClusterManager and feature gate (`api`)

Done: api `aaceeb2d`, `c77b5174`, `4d9027ee`, `1fa143c5` and `dddccc02`. The hub's own trust domain is always accepted (`GetAgentTrustDomains`); every new validation rule and helper is covered by tests.

- [x] Add the `SPIFFERegistration` feature gate (`feature/`).
- [x] Add `authType: spiffe` and the `spiffe` block to the ClusterManager registration drivers
      (`operator/v1/types_clustermanager.go`):
  - `trustDomain`, required: the hub's own trust domain;
  - `agentTrustDomains`, optional: the trust domains the hub accepts agents from (empty means
    only `trustDomain`; the operator fills it);
  - `serverSPIFFEID`, `workloadAPISocket`, `maxSVIDLifetime` (default `24h`) and `maxClockSkew`
    (default `5m`), optional.
- [x] Add `protocol: grpc-spiffe` to `endpointsExposure`.
- [x] Validate that `trustDomain` and each agent trust domain are valid, and that `serverSPIFFEID`
      is a SPIFFE ID. That `serverSPIFFEID` is in `trustDomain` can't be a CEL rule (cost budget,
      see `questions.md` Q4); the operator checks it on Day 14.
- [x] Run `make update` (codegen and CRDs).
- [x] Add `agentIDTemplate` and `addOnAgentIDTemplate` (defaults: the proposal's scheme), with a
      shape pattern and `Get...` helpers. The full checks are the operator's (Day 14). See the
      day document, "Update: agent ID templates", and `questions.md` Q7.

**Done when:** `make verify` passes, and integration tests in `test/integration/api` cover valid
and invalid ClusterManager specs, including the templates.

### Day 3: API, Klusterlet and add-on (`api`)

Done: api `a770a061` and `c77b5174`, pushed to `fork/spire`. The upstream draft PR is not opened
yet.

- [x] Add `authType: spiffe` with `hubEndpoint`, `hubSPIFFEID`, `trustDomain` (the agents' own,
      default: the trust domain of `hubSPIFFEID`) and `workloadAPISocket` to the Klusterlet
      (`operator/v1/types_klusterlet.go`).
- [x] Add `addOnKubeClientRegistrationDriver.authType: spiffe`. It is required when
      `registrationDriver.authType` is `spiffe`; CEL rule added.
- [x] Add the Klusterlet status `status.spiffe.admissionPolicies`: a list keyed by `policyName`,
      each with `policyUID`, `policyGeneration`, `bindingUID` and `bindingGeneration` (a list,
      because recovery tracks three policies).
- [x] Add the Klusterlet status field `status.spiffe.addOnNamespace`: `name` and `uid`.
- [x] Add the condition type constant `SPIFFEAdmissionReady`.
- [x] Add the ManagedCluster annotation constant `open-cluster-management.io/spiffe-trust-domain`.
- [x] Allow `spiffe` as the add-on driver in `ManagedClusterAddOn` `v1beta1` and `v1alpha1`.
- [x] Run `make update` and `make verify` (lint with `DGO_IMAGE=golang:1.26`).
- [x] Add `agentIDTemplate` and `addOnAgentIDTemplate` to the Klusterlet's `spiffe` block, with
      CEL checks for the required placeholders; add `status.spiffe.agentIDs`; require
      `clusterName` with `spiffe`. Coded and tested (247 of 247); not committed yet.

**Done when:** API integration tests pass (202 of 202), the commit is pushed to the fork, and the
`api` PR is opened as a draft.

### Day 4: SPIFFE ID templates and mapping (`sdk-go`, `pkg/spiffe/idtemplate`, `pkg/server/grpc/authn`)

- [ ] Add `github.com/spiffe/go-spiffe/v2` to `go.mod`.
- [ ] Add `pkg/spiffe/idtemplate`: `Compile` (prefix `spiffe://{trustDomain}/`, known and
      required placeholders, whole segments, each once, at least one fixed segment), `Render`,
      `Match` and `Overlaps`.
- [ ] Add `IDScheme` (agent template, add-on template, `agentTrustDomains`) with `Parse`, which maps
      an ID to the user and groups in the proposal's ID-scheme table, plus `system:authenticated`,
      and returns the SVID's trust domain (pinning, Day 11). `{namespace}` is checked and returned
      for logs only.
- [ ] Add `CheckServerID`: the hub's ID must match neither template.
- [ ] Reject IDs from a trust domain not in `agentTrustDomains`, and IDs that match neither template.
- [ ] Write table tests for the templates, and for `Parse` with the default and a custom template.

**Done when:** with the default templates the tests cover every row of the proposal's mapping
table; an ID from `east.example.org` is accepted when listed; an unlisted trust domain, a path like
`.../ocm/something-else` and a generic `/ns/.../sa/...` ID are rejected; and the custom template
`spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}` maps to the same users.

### Day 5: SPIFFE authenticator (`sdk-go`, `pkg/server/grpc/authn`)

- [ ] Add `SPIFFEAuthenticator`, next to `mtls.go` and `token.go`. It reads the verified peer
      SVID from the TLS state, maps it with the `IDScheme` from Day 4, and puts the user, groups, SPIFFE ID and SVID expiry into
      the request context.
- [ ] Add unit and stream interceptors that reject an RPC whose peer SVID has already expired.
- [ ] Write unit tests: a valid SVID, an expired SVID, a missing peer certificate, and a
      certificate without a SPIFFE URI.

**Done when:** the authenticator and its interceptors are tested, and `go vet` passes.

## Week 2: TLS, connections and the hub listener

### Day 6: fake Workload API for tests (`sdk-go`)

- [ ] Write a small in-process Workload API server for tests. It implements the gRPC service from
      `github.com/spiffe/go-spiffe/v2/proto/spiffe/workload` on a unix socket.
- [ ] Have it issue X.509-SVIDs from a test CA with a TTL you choose, and rotate them on demand.
- [ ] Have it return several SVIDs at once, to test SVID picking.
- [ ] Put it in a test helper package that `ocm` can import later.

**Done when:** a test can start the fake server, get an SVID with `workloadapi.X509Source`, rotate
it, and see the new one.

### Day 7: SVID picking and client TLS (`sdk-go`)

- [ ] Add an SVID picker that uses `workloadapi.WithDefaultX509SVIDPicker` and picks by the
      expected SPIFFE ID, never by position. If that ID is missing, it reports not ready and lists
      the IDs it did receive.
- [ ] Add a client option in `pkg/cloudevents/generic/options/grpc` that dials with the agent's SVID:
      `tlsconfig.MTLSClientConfig(source, source, tlsconfig.AuthorizeID(hubSPIFFEID))`.
- [ ] Test the following:
  - the picker picks the right SVID when there are two;
  - the client refuses a server with another ID in the same trust domain;
  - the client picks up a rotated SVID.

**Done when:** these tests pass against the fake Workload API.

### Day 8: server TLS and connection registry (`sdk-go`, `pkg/server/grpc`)

- [ ] Add server TLS from `tlsconfig.MTLSServerConfig`, with an authorizer that accepts any trust
      domain in `agentTrustDomains` (`authorizeMemberOfAny`, from `IDScheme.TrustDomains()`), not
      just the hub's. The credentials wrapper maps peers with `IDScheme.Parse`.
- [ ] Have the registry also close every connection of one trust domain (`CloseTrustDomain`).
- [ ] Add a `TransportCredentials` wrapper. After each handshake it records the peer's SPIFFE ID
      and SVID expiry, and registers the `net.Conn` in a connection registry.
- [ ] Have the registry arm a timer that closes the `net.Conn` when the SVID expires.
- [ ] Have the registry close connections by cluster, and by cluster plus add-on.

**Done when:** a test proves the 10:40 / 11:00 example with a short TTL. The connection is closed
at expiry, and no further unary or stream RPC succeeds on it.

### Day 9: reconnect and renewal (`sdk-go`)

- [ ] Make sure the client reconnects after the server closes the connection, using its current,
      rotated SVID.
- [ ] Test this flow: connect, the SVID rotates, the old SVID expires, the server closes, the
      client reconnects with the new SVID, and a stream resumes.
- [ ] Test that the registry closes connections on demand (deny or delete), and that an RPC already
      running on that connection fails.

**Done when:** the renewal flow works end to end against the fake Workload API. Push the change
and open the `sdk-go` PR as a draft.

### Day 10: SPIFFE listener in the gRPC server (`ocm`, `pkg/server/grpc`)

- [ ] Point `ocm` at the `api` and `sdk-go` fork commits with `replace` lines, then tidy and
      vendor.
- [ ] Add the separate SPIFFE listener. It uses its own port and only the SPIFFE TLS credentials and
      authenticator. It has no CSR or token-request services.
- [ ] When `grpc` is also listed, keep that listener unchanged on its current port.
- [ ] Have both listeners share the same CloudEvents services and authorizer.
- [ ] Add `--spiffe-agent-trust-domains` (default: the hub's trust domain) for the SPIFFE listener.
- [ ] Add `--spiffe-agent-id-template` and `--spiffe-addon-agent-id-template` (defaults: the
      proposal's scheme). Build the `IDScheme` at start and refuse to start on a bad template or a
      server ID that matches one.

**Done when:** the hub starts with only `spiffe`, and with `spiffe` plus `grpc`, and
`go build ./...` passes.

## Week 3: hub authorization, revocation, delete guard

### Day 11: authorization by SVID (`ocm`, `pkg/server/services`)

- [ ] Check every request against the cluster named in the SVID, for both Klusterlet agents and
      add-on agents. An agent of cluster2 can never touch cluster1.
- [ ] For add-on agents, also check the add-on name: the agent of `my-addon` cannot touch
      `other-addon`.
- [ ] Send the mapped groups, including `system:authenticated`, in the SubjectAccessReviews.
- [ ] Never authorize on the `{namespace}` value from the ID template.
- [ ] Pin each cluster to one trust domain: the gRPC cluster service sets
      `open-cluster-management.io/spiffe-trust-domain` from the SVID when the cluster is created
      and keeps it on agent updates; every request is refused unless the SVID's trust domain
      matches it. Without the annotation, one agent trust domain means pinned to it; several mean
      refused until an admin sets it.

**Done when:** unit tests cover cross-cluster and cross-add-on attempts, and both are refused, and
an SVID for cluster2 from `west.example.org` is refused when cluster2 is pinned to
`east.example.org`.

### Day 12: revocation (`ocm`, `pkg/server/grpc`)

- [ ] Watch `ManagedCluster` and `ManagedClusterAddOn`.
- [ ] Close the cluster's connections through the registry when the cluster is denied
      (`hubAcceptsClient: false`) or deleted.
- [ ] Close only that add-on's connections when the add-on is disabled.
- [ ] Refuse new connections from a denied or deleted cluster.

**Done when:** tests cover deny, delete and add-on disable, and each closes exactly the right
connections.

### Day 13: delete guard (`ocm`, `pkg/registration/hub`)

- [ ] Add the finalizer `open-cluster-management.io/spiffe-revocation` to each `ManagedCluster`
      in `spiffe` mode.
- [ ] On delete, keep the finalizer until the deletion time plus `maxSVIDLifetime` plus
      `maxClockSkew`, then remove it. Requeue with the remaining time; don't poll.
- [ ] While the cluster is terminating, deny every request from its identities.

**Done when:** unit tests cover deleted at 10:40 and released at 11:45 (1h lifetime, 5m skew),
including a SPIRE clock that runs ahead and a newer SVID with a longer TTL.

### Day 14: hub operator (`ocm`, `pkg/operator/operators/clustermanager`)

- [ ] In `spiffe` mode, render the gRPC server Deployment with a `csi.spiffe.io` volume.
- [ ] Create no signer Secret, serving certificate or bootstrap token when `spiffe` is the only
      driver.
- [ ] Add the Service `cluster-manager-grpc-server-spiffe` for the `grpc-spiffe` endpoint
      exposure.
- [ ] When `grpc` is removed from the drivers, remove its listener and Secrets.
- [ ] Pass both ID templates to the gRPC server. Check them with `IDScheme` (and the server ID
      with `CheckServerID`); on error, render no SPIFFE listener and set `Degraded`.

**Done when:** manifest tests cover `spiffe` only and `spiffe` plus `grpc`, and no hub Secret
holds key material in `spiffe` only mode.

### Day 15: hub integration test (`ocm`, `test/integration`)

- [ ] Add an integration test with envtest plus the fake Workload API. A fake agent registers, an
      admin accepts the cluster, then the test covers rotation and reconnect, deny, and delete with
      the delete guard.
- [ ] Check that no Secret or ConfigMap with key material or hub credentials is created on the
      hub.
- [ ] Run one registration flow against a second gRPC server with a custom agent ID template.

**Done when:** the new integration test passes, and the existing registration integration suite
still passes.

## Week 4: the agents

### Day 16: spoke registration driver (`ocm`, `pkg/registration/register/spiffe`)

- [ ] Add the new driver next to `csr`, `grpc` and `aws_irsa`. It builds the gRPC clients from
      the SVID and has no CSR step, no Secret and no bootstrap phase.
- [ ] Register it in `register/factory`.
- [ ] Take the agent's own ID from `--spiffe-agent-id` (rendered by the operator, Day 18); don't
      build it in the agent.
- [ ] Make sure the registration agent doesn't start the Secret-based hub-kubeconfig controller in
      `spiffe` mode.

**Done when:** a registration agent with the `spiffe` driver joins the test hub from Day 15.

### Day 17: work agent and singleton agent (`ocm`)

- [ ] Have the work agent fetch its own SVID (role `work`, from `--spiffe-agent-id`) instead of
      reading a Secret.
- [ ] Have the singleton agent use its `klusterlet` ID from `--spiffe-agent-id`.
- [ ] Have each agent report not ready when its expected SVID is missing.

**Done when:** an integration test applies a ManifestWork and syncs its status over the SPIFFE
listener.

### Day 18: Klusterlet operator, four modes (`ocm`, `pkg/operator/operators/klusterlet`)

- [ ] Render the agent Deployments with the CSI volume and the `open-cluster-management.io/spiffe-role`
      label for `Default`, `Singleton`, `Hosted` and `SingletonHosted`.
- [ ] Create no `bootstrap-hub-kubeconfig` and no `hub-kubeconfig-secret` in `spiffe` mode.
- [ ] Render each agent's `--spiffe-agent-id` from the Klusterlet's `agentIDTemplate` (trust
      domain, cluster name, role, and the namespace the pods run in), report the IDs in
      `status.spiffe.agentIDs`, and set `Degraded` on a bad template.
- [ ] In hosted modes, keep the `external-managed-kubeconfig` handling as it is today (Open
      Question 5).

**Done when:** manifest tests cover all four modes, with the right role label and ServiceAccount
pairs from the proposal's table.

### Day 19: admission policy, part 1 (`ocm`, Klusterlet operator)

- [ ] Write the ValidatingAdmissionPolicy and binding for the agent namespaces:
  - only the operator may create or change Deployments;
  - ReplicaSets come only from the Deployment controller, owned by a Deployment, and nobody may
    change a ReplicaSet's Pod template;
  - Pods come only from the ReplicaSet controller, owned by a ReplicaSet;
  - StatefulSets, DaemonSets, Jobs and CronJobs are refused;
  - each Pod's `spiffe-role` label and ServiceAccount must be an allowed pair, and neither may
    change after creation.
- [ ] Have the operator check that the policy API is served and Kubernetes is 1.30 or later. If
      not, it deploys no agents and sets `Degraded`.

**Done when:** envtest tests cover every allowed and refused case. envtest must run Kubernetes
1.30 or later; check the envtest version first.

### Day 20: admission policy, part 2 (`ocm`)

- [ ] After creation, nobody may change a Pod's container images or add ephemeral containers.
- [ ] Refuse `CONNECT` on `pods/exec` and `pods/attach`, and still allow `pods/log`.
- [ ] Write the add-on namespace rules:
  - only the work agent may create workloads;
  - DaemonSets and StatefulSets are allowed;
  - add-on pods use the `addon` role;
  - Klusterlet ServiceAccounts and roles are refused.
- [ ] Write the namespace label rules:
  - `open-cluster-management.io/spiffe-addon-namespace: <klusterlet>` may be set only at creation,
    by the work agent or the operator;
  - nobody may add or change it later, and only the operator may remove it;
  - in a namespace labeled for one Klusterlet, only that Klusterlet's work agent may create
    workloads.

**Done when:** tests cover exec and attach refused even for a namespace `admin`, label changes
refused, and the hosted case with two Klusterlets.

## Week 5: recovery, trusted state, add-ons

### Day 21: recovery and trusted state (`ocm`, Klusterlet operator)

- [ ] Write the recovery steps:
  1. Scale the operator's agent Deployments to zero and wait for their Pods to go.
  2. Recreate the policy and binding, note their UIDs and generations, and remove this
     Klusterlet's label from any namespace that isn't on record.
  3. Delete every workload in the agent namespaces and the recorded add-on namespaces, and wait
     until no Pod is left.
  4. Compare the policy and binding with the noted values and restart if anything changed.
     Otherwise record them, set `SPIFFEAdmissionReady`, and scale the agents up.
- [ ] Have every reconcile, including after an operator restart, compare the live policy and
      binding with `status.spiffe.admissionPolicy`, and run recovery on any difference.
- [ ] Treat a namespace as recorded only if its UID is in one of this Klusterlet's
      `AppliedManifestWork`s, or matches `status.spiffe.addOnNamespace`.
- [ ] Never label an existing namespace.

**Done when:** the proposal's recovery and trusted-state tests pass:
- first activation;
- the policy recreated, or edited, while the operator was down;
- the policy edited during recovery;
- a DaemonSet add-on removed during recovery and re-applied afterwards;
- a fake label on `default` is removed without anything there being deleted;
- an operator upgrade that changes the policy runs no recovery.

### Day 22: add-on registration (`ocm`, `pkg/registration/spoke/addon` and `pkg/addon`)

- [ ] On the spoke, with the `spiffe` add-on driver, create no CSR, token or
      `<addon>-hub-kubeconfig` Secret. Set `kubeClient.driver: spiffe` and `RegistrationApplied`.
- [ ] Refuse a `customSigner` registration with a clear condition.
- [ ] Set `AddOnNamespaceNotDedicated` when the install namespace lacks the label.
- [ ] On the hub, have the add-on manager check that the `kubeClient` subject can be carried by an
      SVID. The default subject passes; a custom user such as `my-team-bot` gets
      `RegistrationApplied=False`.

**Done when:** unit tests cover each case, and an integration test enables, rotates and disables
an add-on. Push the change and open the `ocm` PR as a draft.

### Day 23: addon-framework

- [ ] Point it at the fork commits with `replace` lines.
- [ ] Add a helper that builds an add-on agent's gRPC clients from its SVID, rendering its ID from
      `addOnAgentIDTemplate` (how the template reaches the agent: `questions.md` Q8).
- [ ] When the cluster uses the `spiffe` add-on driver, render Deployments, DaemonSets and
      StatefulSets with the CSI volume and `spiffe-role: addon`, instead of the
      `<addon>-hub-kubeconfig` Secret volume.
- [ ] Add the `spiffe-addon-namespace: <klusterlet>` label to the add-on's namespace manifest.

**Done when:** unit tests cover all three workload kinds and the namespace label. Open the
`addon-framework` PR as a draft.

### Day 24: clusteradm (`clusteradm`, `pkg/cmd/init`, `join` and `accept`)

- [ ] `init`: add `--registration-drivers spiffe`, `--spiffe-trust-domain` and
      `--spiffe-agent-trust-domains`.
- [ ] `join`: add `--registration-auth spiffe`, `--hub-grpc-endpoint`, `--hub-spiffe-id` and
      `--spiffe-trust-domain` (the agents' own), with no token. Wait for `SPIFFEAdmissionReady`
      and say so.
- [ ] `accept`: no CSR to approve in `spiffe` mode. `--spiffe-trust-domain` confirms the
      cluster's pinned trust domain, or sets it when missing.
- [ ] Make `--singleton` and `--mode hosted` work with `spiffe`.
- [ ] `init` and `join`: add `--spiffe-agent-id-template` and `--spiffe-addon-agent-id-template`,
      checked locally. `join` requires `--cluster-name` with `spiffe` and prints the agent IDs the
      platform must issue.

**Done when:** unit tests cover flag validation and the rendered Klusterlet for each mode. Open the
`clusteradm` PR as a draft.

### Day 25: migration from `grpc` (`ocm`)

- [ ] Test `spiffe` and `grpc` listed together: a `grpc` agent and a `spiffe` agent both work.
- [ ] Move one cluster from `grpc` to `spiffe`. It keeps its ManagedCluster, ManifestWorks,
      add-ons and RBAC, and the old Secrets are deleted after the switch.
- [ ] Test the downgrade path.

**Done when:** the migration integration test passes.

## Week 6: end to end and review

### Day 26: e2e environment

- [ ] Write a script that creates kind clusters with SPIRE from the SPIFFE Helm charts: `spire-crds`
      0.6.1 and `spire` 0.30.4, matching ocm-hub.
- [ ] Use `ClusterSPIFFEID`s for the gRPC server, the agents and one add-on, following
      `plan/clusterspiffeid.md` (with `className`), plus a second set for a custom ID template.
- [ ] Cover both cases: a spoke in the hub's trust domain, and a spoke in its own trust domain
      (`east.example.org`) federated with the hub's. This is open question 4: pick the reference
      SPIRE topology here and record it.

**Done when:** a hub and two managed clusters come up with SPIRE, the federated one sees the hub's
bundle, and an agent pod on each gets its SVID.

### Day 27: e2e, Default and Singleton

- [ ] Run join, accept, ManifestWork apply and status, add-on enable and status sync, SVID rotation
      under load, cluster revocation and single add-on revocation.
- [ ] Switch a live cluster to a custom agent ID template and check it recovers on its own.

**Done when:** both modes pass.

### Day 28: e2e, Hosted and SingletonHosted

- [ ] Run the same flows with a management cluster.
- [ ] Test two Klusterlets on one management cluster, and recovery of one leaving the other
      running.
- [ ] Use the custom template, so `{namespace}` is the Klusterlet's namespace on the management
      cluster.

**Done when:** both modes pass.

### Day 29: real cluster check

- [ ] Start the AKS clusters `ocm-hub` and `ocm-spoke` in resource group `ocm-spiffe-rg` (they
      are stopped).
- [ ] Repeat Day 27 there, including an agent that was down past its SVID's expiry and comes back.
- [ ] Confirm kube-controller-manager uses separate controller ServiceAccounts, which the admission
      policy needs. Check this rather than assuming it.
- [ ] Stop the clusters afterwards.

**Done when:** the flows pass on AKS, or every difference is written down.

### Day 30: clean up and open PRs for review

- [ ] Run `make verify` and the full unit and integration suites in every repo.
- [ ] Mark the PRs ready, in order: `api`, `sdk-go`, `ocm`, `addon-framework`, `clusteradm`. In
      each, link the enhancement PR #203 and the PRs it depends on.
- [ ] Update the proposal's editable sections for the agent ID templates (API, hub setup, Open
      Question 1, Risks); raise the frozen ID-scheme table with the reviewers.
- [ ] Update the proposal's Implementation History with the PR links. That section comes after
      the frozen part, so it can be edited.

**Done when:** all PRs are open for review, CI is green, and the remaining work is listed below.

## Buffer and follow-ups

Plan for about one buffer day a week. Admission policy (Days 19 to 21) and e2e (Days 26 to 28) are
the most likely to overrun.

Left open by the proposal; don't build these without a decision:

- Open question 1 is decided: the ID layout is configurable with agent ID templates
  (`questions.md` Q7).
- Open question 2: should clusters be auto-accepted?
- Open question 3: should the agent create the `ManagedCluster`, or the admin?
- Open question 5: replacing the `external-managed-kubeconfig` in hosted modes.
- Open question 6: extra gRPC services for add-ons that call hub APIs directly.

## Daily routine

1. Pull upstream `main` into `spire` if it moved, and fix any conflicts first.
2. Do the day's checklist. Commit small, signed-off commits (`git commit -s`).
3. Before stopping: `go build ./...`, `go vet ./...` and the unit tests for the packages you
   touched.
4. Tick the boxes in this file, and note anything that slipped to the next day.
