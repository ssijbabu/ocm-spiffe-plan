# Day 17: work agent and singleton agent

**Goal:** the work agent fetches its own SVID (`agent/work`) and applies ManifestWorks over the
SPIFFE listener. The singleton agent uses one SVID (`agent/klusterlet`) for both jobs.

**Repo:** `ocm`, branch `spire`, packages `pkg/work/spoke`, `pkg/singleton`, `pkg/cmd/spoke`.

## Before you start

| Input | Where |
| --- | --- |
| How the work agent gets hub access with `grpc` | `manifests/klusterlet/management/klusterlet-work-deployment.yaml`: with `authType: grpc` it gets `--cloudevents-client-id=<cluster>-work-agent` and reads a gRPC config file |
| Work agent options | `pkg/work/spoke/options.go` (check the real file), `--workload-source-driver`, `--workload-source-config` |
| Singleton agent | `pkg/singleton` |

Today, with `grpc`, the registration agent writes the hub credentials to a Secret and the work agent
reads them from a mounted file. In `spiffe` mode the work agent has its own SVID, so nothing is
shared through a Secret.

## Task 1: work agent flags

Add the same three flags as the registration agent:

```text
--spiffe-hub-endpoint=ocm-grpc.hub.internal:443
--spiffe-hub-id=spiffe://ocm-hub.example.org/ocm/hub/grpc-server
--spiffe-workload-api-socket=unix:///spiffe-workload-api/spire-agent.sock
```

When `--spiffe-hub-endpoint` is set, the work agent builds its CloudEvents client with
`NewSPIFFEGRPCOptions` and the agent ID `spiffe://<td>/ocm/cluster/<cluster>/agent/work`, instead
of loading `--workload-source-config`.

```go
func (o *WorkloadAgentOptions) cloudEventsOptions(ctx context.Context) (*grpcoptions.GRPCOptions, io.Closer, error) {
	if o.SPIFFE.HubEndpoint == "" {
		return grpcoptions.BuildGRPCOptionsFromFlags(o.WorkloadSourceConfig) // unchanged path, no closer
	}
	return grpcoptions.NewSPIFFEGRPCOptions(ctx, grpcoptions.SPIFFEConfig{
		URL:               o.SPIFFE.HubEndpoint,
		HubSPIFFEID:       o.SPIFFE.HubSPIFFEID,
		AgentSPIFFEID:     o.SPIFFE.AgentID(o.SpokeClusterName, "work"),
		WorkloadAPISocket: o.SPIFFE.WorkloadAPISocket,
	})
}
```

Share the `Option` type from Day 16 (`pkg/registration/register/spiffe`) or move it to a small
common package (for example `pkg/common/spiffe`) so both agents use it.

## Task 2: singleton agent

The singleton agent runs registration and work in one process with the role `klusterlet`. Pass
`role = "klusterlet"` to both, so both use
`spiffe://<td>/ocm/cluster/<cluster>/agent/klusterlet`, and open the Workload API source once and
share it (two `X509Source`s to the same socket would work, but one is cleaner).

## Task 3: readiness

Each agent's health check calls `GetX509SVID()` on its source and reports not ready with the
`ErrSVIDNotFound` message when the SVID is missing. Check how the agents expose health today
(`/healthz` handlers in `controllercmd`) and add a checker.

Example readiness output when SPIRE has no entry for the pod:

```text
$ kubectl -n open-cluster-management-agent get pod -l app=klusterlet-manifestwork-agent
NAME                                READY   STATUS    RESTARTS
klusterlet-work-agent-6f9c7-x2kqp   0/1     Running   0
$ kubectl -n open-cluster-management-agent logs deploy/klusterlet-work-agent | tail -1
E1120 10:02:11 health.go:41] not ready: no SVID with ID spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work from the Workload API; received []
```

## Task 4: tests

**Unit:**

| Case | Input | Expected |
| --- | --- | --- |
| Work agent ID | cluster `cluster1` | `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work` |
| Singleton ID | singleton, cluster `cluster1` | `.../agent/klusterlet` for both registration and work |
| No SPIFFE flags | existing config file | old path unchanged |

**Integration:** new spec in `test/integration/work` (check where the gRPC work tests are,
`test-cloudevents-work-grpc-integration` in `test/integration-test.mk`):

| Step | Input | Expected |
| --- | --- | --- |
| 1 | work agent with SVID `.../cluster1/agent/work` | connects to the SPIFFE listener |
| 2 | hub creates a ManifestWork with a ConfigMap `cm1` in `default` for cluster1 | `cm1` appears on the spoke (envtest) |
| 3 | spoke changes `cm1`'s status-feedback field | the ManifestWork status on the hub shows it |
| 4 | work agent with SVID `.../cluster2/agent/work` | can't read cluster1's ManifestWorks |

**Run:**

```shell
dgo 'go test ./pkg/work/... ./pkg/singleton/... -run SPIFFE -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'make test-cloudevents-work-grpc-integration ARGS="-ginkgo.focus=spiffe"' 2>&1 | tail -2
```

## Task 5: commit

```shell
git add -A && git commit -s -m ":sparkles: work and singleton agents connect with their own SVIDs"
```

## Rules check

- [ ] The work agent doesn't read anything written by the registration agent.
- [ ] Each agent presents only its own role's SVID.

## Update: multiple agent trust domains

The work and singleton agents also take `--spiffe-trust-domain` and build their IDs in it.
