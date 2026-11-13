# Day 9: renewal end to end, and the sdk-go draft PR

**Goal:** prove that an agent keeps working across SVID rotation: the server closes the old
connection at expiry, the client reconnects with its new SVID, and a CloudEvents stream resumes.
Then push sdk-go and open its draft PR.

**Repo:** `sdk-go`, branch `spire`.

## Before you start

| Input | Where |
| --- | --- |
| Client options | Day 7, `NewSPIFFEGRPCOptions` |
| Server credentials and registry | Day 8 |
| CloudEvents gRPC client | `pkg/cloudevents/generic/options/grpc/agentoptions.go` and `protocol/` |

The renewal sequence to test:

```text
10:00  agent SVID A issued, valid to 11:00; agent connects, opens a subscribe stream
10:30  SPIRE rotates: agent's source now has SVID B, valid to 11:30
       (the open connection keeps using A: TLS identity is fixed per connection)
11:00  server closes the connection opened with A
11:00+ client reconnects; the new handshake uses B; the stream is re-opened
```

## Task 1: check how the client reconnects

**Input:** the CloudEvents gRPC client code.

**Steps:** read how the client reacts when the connection drops (look for reconnect, backoff and
resubscribe logic). Write down:

- whether it reconnects by itself, or relies on gRPC's channel reconnect;
- whether it re-sends the subscribe request after reconnecting;
- the backoff (first retry delay).

**Output:** a few lines in `plan/questions.md`. If it does not re-subscribe on its own, the fix
belongs in this repo today, with a test.

## Task 2: the renewal test

**New file:** `pkg/cloudevents/generic/options/grpc/spiffe_renewal_test.go`. Use short TTLs so it
runs in seconds:

```go
func TestSPIFFERenewal(t *testing.T) {
	ca := fakespire.NewCA(t, "ocm-hub.example.org")
	hubAPI, agentAPI := fakespire.Start(t, ca), fakespire.Start(t, ca)
	hubAPI.SetSVIDs(ca.Issue(t, "spiffe://ocm-hub.example.org/ocm/hub/grpc-server", time.Hour))
	agentAPI.SetSVIDs(ca.Issue(t, workID, 3*time.Second))           // SVID A

	reg := spiffe.NewRegistry()
	addr := startTestBroker(t, hubAPI, reg)                          // gRPC CloudEvents broker with SPIFFE creds

	opts, closer, err := NewSPIFFEGRPCOptions(ctx, SPIFFEConfig{
		URL: addr, HubSPIFFEID: hubID, AgentSPIFFEID: workID, WorkloadAPISocket: agentAPI.Addr,
	})
	require.NoError(t, err)
	defer closer.Close()

	events := subscribe(t, opts)                                     // returns a channel of received events
	publishFromHub(t, "e1"); requireEvent(t, events, "e1")

	agentAPI.SetSVIDs(ca.Issue(t, workID, time.Hour))                // rotate to SVID B
	time.Sleep(4 * time.Second)                                      // A expires; server closes

	publishFromHub(t, "e2"); requireEvent(t, events, "e2")           // resumed on the new connection
	require.Equal(t, serialOf(B), reg.PeerSerial(workID))            // server saw SVID B
}
```

`startTestBroker`, `publishFromHub` and `subscribe` are small helpers in the test file; look at
the existing broker tests in `pkg/cloudevents/server/grpc` for how they build a broker.

| Step | Input | Expected output |
| --- | --- | --- |
| First event | hub publishes `e1` | agent receives `e1` over the connection made with SVID A |
| Rotation | `SetSVIDs(B)` | nothing visible; connection still uses A |
| Expiry | 3s pass | server log: `closing connection of spiffe://.../agent/work: SVID expired at ...` |
| Second event | hub publishes `e2` | agent receives `e2`; server's tracked connection shows SVID B |

## Task 3: revocation while streaming

| Case | Input | Expected output |
| --- | --- | --- |
| Deny | stream open; `reg.CloseCluster("cluster1")` | stream ends; the client's reconnect is accepted at TLS level (the authorizer refuses it later, Day 12) |
| Running unary call | slow unary handler running; `CloseCluster` | the call returns `Unavailable` |

## Task 4: run the whole package suite

```shell
dgo 'make test' 2>&1 | tail -5
grep -hE '^(FAIL|--- FAIL)' <log> | sort | uniq > /tmp/now.txt
diff ../plan/baseline/known-failures.txt /tmp/now.txt
```

**Output:** no new failures compared to the Day 1 baseline.

## Task 5: push and open the draft PR

```shell
git push fork spire
gh pr create -R open-cluster-management-io/sdk-go --draft --head ssijbabu:spire \
  --title ":sparkles: SPIFFE authentication for the gRPC server and agents" \
  --body "Implements the sdk-go part of enhancement #199 (open-cluster-management-io/enhancements#203):
SPIFFE ID mapping, SPIFFEAuthenticator, SVID picking, SPIFFE client and server TLS, and closing
connections at SVID expiry. Depends on the api PR for nothing; ocm uses it."
git rev-parse HEAD
```

**Output:** draft PR URL and the head SHA, which `ocm` will `replace` to on Day 10.

## End of day

| Output | Check |
| --- | --- |
| Renewal and revocation tests | pass with `-race` |
| sdk-go draft PR | open, CI running |

## Rules check

- [ ] Renewal needs no OCM action: SPIRE rotates, the connection is replaced.
- [ ] No step writes the SVID anywhere.

## Result (2026-11-13)

Tasks 1–4 done: sdk-go `7541964e`, pushed to `fork/spire` with Days 4–8. Task 5: pushed, but the draft PR is not opened yet.

- Task 1: see `questions.md` Q9.
- Tasks 2–3: `pkg/cloudevents/generic/options/v2/grpc/spiffe_renewal_test.go` (the v2 transport is
  what agents use). The test drives the transport the way the base client does (close, wait,
  connect, subscribe, receive), with a 100ms delay instead of `DelayFn`. Renewal, revocation while
  streaming and a running unary call ending with `Unavailable` all pass with `-race`.
- Task 4: `make test-unit`: 56 packages ok, 0 failures (baseline 52; the 4 new packages are
  `pkg/spiffe`, `pkg/spiffe/idtemplate`, `pkg/server/grpc/spiffe`, `pkg/testing/fakespire`).
  `make verify` (Go 1.26, golangci-lint v2.12.2): 0 issues. Same result at the pushed HEAD, after
  switching `api` to the fork commit.
- sdk-go head for `ocm`'s `replace` (Day 10): `7541964e`.
