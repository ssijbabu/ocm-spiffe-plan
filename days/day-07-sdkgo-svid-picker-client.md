# Day 7: SVID picking and the client TLS option

**Goal:** an agent picks its SVID by the ID it expects (never by position), and dials the hub over
mTLS, accepting only the hub's exact SPIFFE ID.

**Repo:** `sdk-go`, branch `spire`.

## Before you start

| Input | Where |
| --- | --- |
| Fake Workload API | Day 6, `pkg/testing/fakespire` |
| Client dialer | `pkg/cloudevents/generic/options/grpc/options.go`: `GRPCDialer{URL, TLSConfig, ...}`, `GRPCOptions{Dialer}`, `BuildGRPCOptionsFromFlags(configPath)` |

Today the client loads a YAML file (`GRPCConfig`) with a URL, client certificate and CA, or a
token. In `spiffe` mode there is no file with credentials: the TLS config comes from the Workload
API.

## Task 1: the SVID picker

**New file:** `pkg/spiffe/source.go` (a new small package, shared by server and client).

```go
package spiffe

// ErrSVIDNotFound is returned when the Workload API has no SVID with the expected ID.
type ErrSVIDNotFound struct {
	Want spiffeid.ID
	Got  []spiffeid.ID
}

func (e *ErrSVIDNotFound) Error() string {
	return fmt.Sprintf("no SVID with ID %s from the Workload API; received %v", e.Want, e.Got)
}

// NewX509Source opens the Workload API at addr and always presents the SVID whose ID is want.
// It blocks until the first update arrives or ctx ends.
func NewX509Source(ctx context.Context, addr string, want spiffeid.ID) (*workloadapi.X509Source, error) {
	var missing atomic.Pointer[ErrSVIDNotFound]
	picker := func(svids []*x509svid.SVID) *x509svid.SVID {
		got := make([]spiffeid.ID, 0, len(svids))
		for _, s := range svids {
			if s.ID == want {
				missing.Store(nil)
				return s
			}
			got = append(got, s.ID)
		}
		missing.Store(&ErrSVIDNotFound{Want: want, Got: got})
		return nil
	}
	src, err := workloadapi.NewX509Source(ctx,
		workloadapi.WithClientOptions(workloadapi.WithAddr(addr)),
		workloadapi.WithDefaultX509SVIDPicker(picker))
	if err != nil {
		return nil, err
	}
	if e := missing.Load(); e != nil {
		src.Close()
		return nil, e
	}
	return src, nil
}
```

Check what `X509Source.GetX509SVID` does when the picker returns `nil` in the go-spiffe version
you vendored. If it panics or returns a stale SVID, wrap the source in a small type whose
`GetX509SVID` returns `ErrSVIDNotFound` instead, and use that wrapper everywhere.

Readiness: the component's readiness check calls `GetX509SVID()` and reports not ready with the
error message, for example:

```text
not ready: no SVID with ID spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work from the
Workload API; received [spiffe://ocm-hub.example.org/ns/open-cluster-management-agent/sa/default]
```

## Task 2: the client option

**Add to** `pkg/cloudevents/generic/options/grpc/options.go`:

```go
// SPIFFEConfig is what an agent needs to dial the hub with its SVID. Nothing in it is secret.
type SPIFFEConfig struct {
	URL               string // host:port of the hub's grpc-spiffe listener
	HubSPIFFEID       string // the only server ID accepted
	AgentSPIFFEID     string // the SVID to present
	WorkloadAPISocket string
}

// NewSPIFFEGRPCOptions builds client options whose TLS config comes from the Workload API.
// The returned source must be closed when the agent stops.
func NewSPIFFEGRPCOptions(ctx context.Context, c SPIFFEConfig) (*GRPCOptions, io.Closer, error) {
	hubID, err := spiffeid.FromString(c.HubSPIFFEID)
	if err != nil {
		return nil, nil, fmt.Errorf("invalid hub SPIFFE ID: %w", err)
	}
	agentID, err := spiffeid.FromString(c.AgentSPIFFEID)
	if err != nil {
		return nil, nil, fmt.Errorf("invalid agent SPIFFE ID: %w", err)
	}
	src, err := ocmspiffe.NewX509Source(ctx, c.WorkloadAPISocket, agentID)
	if err != nil {
		return nil, nil, err
	}
	opts := NewGRPCOptions()
	opts.Dialer.URL = c.URL
	opts.Dialer.TLSConfig = tlsconfig.MTLSClientConfig(src, src, tlsconfig.AuthorizeID(hubID))
	return opts, src, nil
}
```

`MTLSClientConfig` uses callbacks (`GetClientCertificate`, `VerifyPeerCertificate`), so every new
handshake uses the current SVID and bundle. Rotation needs no extra code on the client.

## Task 3: tests

**New file:** `pkg/spiffe/source_test.go` and a test in the grpc options package.

Test setup: one fake Workload API for the agent, a second for the hub (same CA), and a minimal
gRPC server with `tlsconfig.MTLSServerConfig` and the gRPC health service.

| Case | Input | Expected output |
| --- | --- | --- |
| Pick by ID | Workload API returns `[ns/.../sa/default, .../agent/work]`; want `.../agent/work` | the source presents `.../agent/work` |
| Missing ID | returns only `ns/.../sa/default` | `NewX509Source` returns `*ErrSVIDNotFound` with `Got` = `[ns/.../sa/default]` |
| Right hub | server presents `spiffe://ocm-hub.example.org/ocm/hub/grpc-server` | health check returns `SERVING` |
| Wrong hub, same trust domain | server presents `spiffe://ocm-hub.example.org/ocm/something-else` | dial fails; error contains `unexpected ID` |
| Rotation | after one call, `SetSVIDs` with a new agent SVID; open a new connection | server sees the new serial number |

Example for the "wrong hub" case:

```go
_, err := healthpb.NewHealthClient(conn).Check(ctx, &healthpb.HealthCheckRequest{})
require.Error(t, err)
require.Contains(t, err.Error(), `unexpected ID "spiffe://ocm-hub.example.org/ocm/something-else"`)
```

**Run:**

```shell
dgo 'go test ./pkg/spiffe/... ./pkg/cloudevents/generic/options/grpc/... -run "Picker|SPIFFE" -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

**Output:**

```text
--- PASS: TestNewX509SourcePicksByID (0.06s)
--- PASS: TestNewX509SourceMissingID (0.03s)
--- PASS: TestSPIFFEClientAcceptsOnlyHubID (0.11s)
--- PASS: TestSPIFFEClientUsesRotatedSVID (0.21s)
ok  	open-cluster-management.io/sdk-go/pkg/spiffe	0.140s
ok  	open-cluster-management.io/sdk-go/pkg/cloudevents/generic/options/grpc	0.512s
```

## Task 4: commit

```shell
git add -A && git commit -s -m ":sparkles: pick SVIDs by ID and dial the hub with SPIFFE mTLS"
```

## End of day

| Output | Used by |
| --- | --- |
| `spiffe.NewX509Source(ctx, addr, want)` | server (Day 8), agents (Days 16, 17), add-on helper (Day 23) |
| `NewSPIFFEGRPCOptions` | agents (Days 16, 17) |

## Rules check

- [ ] The SVID and key exist only in memory (inside `X509Source`); nothing is written to disk.
- [ ] The client accepts only the configured hub ID, not any ID in the trust domain.
