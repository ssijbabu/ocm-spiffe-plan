# Day 30: clean up and open the PRs for review

**Goal:** every repo passes its checks, the PRs are ready in dependency order, and the proposal
links them.

## Before you start

| Input | Where |
| --- | --- |
| Draft PRs | api (Day 3), sdk-go (Day 9), ocm (Day 22), addon-framework (Day 23), clusteradm (Day 24) |
| Results | `plan/e2e-results.md`, `plan/aks-results.md` |

## Task 1: full checks in every repo

```shell
cd /home/ssijbabu/Documents/repos/ocm
(cd api && dgo 'make verify && make test-api-integration')
(cd sdk-go && dgo 'make verify && make test')
(cd ocm && dgo 'make verify && make test-unit && make test-integration')
(cd addon-framework && dgo 'make verify && make test')
(cd clusteradm && dgo 'make verify && make test')
```

**Output:** each exits 0. Compare failures with `plan/baseline/known-failures.txt`; only the known
ones may remain.

## Task 2: the rules, one last time

Go through the code, not the plan:

| Rule (CLAUDE.md) | How to check |
| --- | --- |
| Only SVIDs for registration, renewal, revocation | `grep -rn 'spiffe' --include=*.go` in the driver paths: no CSR, token or Secret use |
| OCM never signs | no new code under `certrotation` or signer paths is reached in `spiffe` mode |
| SVIDs in memory only | no `os.WriteFile`, no Secret/ConfigMap create in the `spiffe` paths |
| Only the local Workload API | the only new outbound connection is the hub endpoint (agents) and the Workload API socket |
| Four modes | e2e table complete |
| Add-ons | e2e add-on rows pass |
| Feature gate | with `SPIFFERegistration` off, `authType: spiffe` is refused or ignored with a clear condition |

## Task 3: rebase and order

Rebase each `spire` on its upstream `main`. Then, in order:

1. **api:** mark ready. Once merged, note the version or pseudo-version.
2. **sdk-go:** switch its `api` dependency to the merged version (if it uses the new types), mark
   ready.
3. **ocm:** drop the `replace` lines for merged repos, `go mod tidy && go mod vendor`, mark ready.
4. **addon-framework:** same.
5. **clusteradm:** same; it depends on `ocm` too.

PR description template:

```markdown
Implements part of open-cluster-management-io/enhancements#199 (proposal PR #203).

Depends on: open-cluster-management-io/api#<n>, open-cluster-management-io/sdk-go#<n>

What this changes:
- ...

Tests:
- unit: ...
- integration: ...
- e2e: kind + SPIRE 1.15.5, all four install modes (see results table)

Not in this PR:
- Open questions 1, 2, 3, 5, 6 of the proposal.
```

## Task 4: update the proposal

On the enhancements `spire` branch, edit **Implementation History** (after the frozen part):

```markdown
## Implementation History

- 2026-11-03: proposal opened (#203).
- 2026-12-xx: implementation PRs opened: api#<n>, sdk-go#<n>, ocm#<n>, addon-framework#<n>, clusteradm#<n>.
```

Check the frozen part is unchanged before committing:

```shell
cd enhancements && git diff -U0 HEAD -- enhancements/sig-architecture/199-spiffe-grpc-registration/README.md | head
```

## Task 5: list what's left

**Output:** `plan/follow-ups.md` with anything found during the work and the proposal's open
questions:

```markdown
- [ ] OQ1 SPIFFE ID scheme configurable?
- [ ] OQ2 auto-accept by SVID pattern?
- [ ] OQ3 agent-created vs admin-created ManagedCluster
- [ ] OQ5 replace external-managed-kubeconfig in hosted modes
- [ ] OQ6 extra gRPC services for add-ons that call hub APIs
- [ ] AKS controller identity result (Day 29)
```

## End of the plan

| Output | Check |
| --- | --- |
| Five PRs ready for review | CI green |
| Proposal links them | Implementation History updated |
| Follow-ups | `plan/follow-ups.md` |
