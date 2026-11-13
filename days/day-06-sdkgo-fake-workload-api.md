# Day 6: fake SPIRE Workload API for tests

**Goal:** an in-process Workload API server that issues X.509-SVIDs from a test CA, so unit and
integration tests (here and in `ocm`) can run without SPIRE.

**Repo:** `sdk-go`, branch `spire`, new package `pkg/testing/fakespire`.

## Before you start

| Input | Where |
| --- | --- |
| Workload API gRPC service | `github.com/spiffe/go-spiffe/v2/proto/spiffe/workload` (vendored on Day 4) |
| Client used by OCM | `workloadapi.NewX509Source` |

How the real Workload API works, in short:

- It is a gRPC service on a unix socket (`unix:///spiffe-workload-api/spire-agent.sock` in OCM pods).
- `FetchX509SVID` is a server stream. The first message carries the current SVIDs, and a new
  message is sent whenever they rotate.
- Every request must carry the metadata header `workload.spiffe.io: true`; the server refuses
  calls without it.
- An `X509SVID` message holds `spiffe_id`, `x509_svid` (the certificate chain as concatenated
  DER), `x509_svid_key` (PKCS#8 DER) and `bundle` (the trust bundle as concatenated DER).

go-spiffe's own fake server is in an `internal` package, so OCM can't import it. Write a small
one.

## Task 1: the test CA

**New file:** `pkg/testing/fakespire/ca.go`

```go
package fakespire

// CA is an in-memory SPIFFE trust domain CA for tests.
type CA struct {
	TrustDomain spiffeid.TrustDomain
	cert        *x509.Certificate
	key         crypto.Signer
}

func NewCA(t testing.TB, td string) *CA {
	key, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	tdID := spiffeid.RequireTrustDomainFromString(td)
	tmpl := &x509.Certificate{
		SerialNumber:          big.NewInt(1),
		NotBefore:             time.Now().Add(-time.Hour),
		NotAfter:              time.Now().Add(24 * time.Hour),
		IsCA:                  true,
		BasicConstraintsValid: true,
		KeyUsage:              x509.KeyUsageCertSign | x509.KeyUsageCRLSign,
		URIs:                  []*url.URL{tdID.ID().URL()},
	}
	der, err := x509.CreateCertificate(rand.Reader, tmpl, tmpl, &key.PublicKey, key)
	require.NoError(t, err)
	cert, _ := x509.ParseCertificate(der)
	return &CA{TrustDomain: tdID, cert: cert, key: key}
}

// Issue creates an X.509-SVID for id, valid for ttl.
func (c *CA) Issue(t testing.TB, id string, ttl time.Duration) *x509svid.SVID {
	key, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	tmpl := &x509.Certificate{
		SerialNumber: big.NewInt(time.Now().UnixNano()),
		NotBefore:    time.Now().Add(-time.Second),
		NotAfter:     time.Now().Add(ttl),
		KeyUsage:     x509.KeyUsageDigitalSignature | x509.KeyUsageKeyEncipherment,
		ExtKeyUsage:  []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth, x509.ExtKeyUsageClientAuth},
		URIs:         []*url.URL{spiffeid.RequireFromString(id).URL()},
	}
	der, err := x509.CreateCertificate(rand.Reader, tmpl, c.cert, &key.PublicKey, c.key)
	require.NoError(t, err)
	cert, _ := x509.ParseCertificate(der)
	return &x509svid.SVID{ID: spiffeid.RequireFromString(id), Certificates: []*x509.Certificate{cert}, PrivateKey: key}
}

// Bundle is the trust bundle for this CA.
func (c *CA) Bundle() *x509bundle.Bundle {
	return x509bundle.FromX509Authorities(c.TrustDomain, []*x509.Certificate{c.cert})
}
```

## Task 2: the server

**New file:** `pkg/testing/fakespire/server.go`

```go
// Server is a fake Workload API. Set SVIDs with SetSVIDs; every open FetchX509SVID
// stream receives the update, like a SPIRE agent after rotation.
type Server struct {
	workload.UnimplementedSpiffeWorkloadAPIServer
	ca     *CA
	mu     sync.Mutex
	svids  []*x509svid.SVID
	notify chan struct{}
	Addr   string // "unix:///tmp/.../agent.sock", pass to workloadapi.WithAddr
}

func Start(t testing.TB, ca *CA) *Server {
	dir := t.TempDir()
	sock := filepath.Join(dir, "agent.sock")
	lis, err := net.Listen("unix", sock)
	require.NoError(t, err)
	s := &Server{ca: ca, notify: make(chan struct{}), Addr: "unix://" + sock}
	gs := grpc.NewServer()
	workload.RegisterSpiffeWorkloadAPIServer(gs, s)
	go func() { _ = gs.Serve(lis) }()
	t.Cleanup(gs.Stop)
	return s
}

// SetSVIDs replaces the SVIDs and pushes them to every watcher.
func (s *Server) SetSVIDs(svids ...*x509svid.SVID) {
	s.mu.Lock()
	s.svids = svids
	close(s.notify)
	s.notify = make(chan struct{})
	s.mu.Unlock()
}

func (s *Server) FetchX509SVID(_ *workload.X509SVIDRequest, stream workload.SpiffeWorkloadAPI_FetchX509SVIDServer) error {
	if md, _ := metadata.FromIncomingContext(stream.Context()); len(md.Get("workload.spiffe.io")) == 0 {
		return status.Error(codes.InvalidArgument, "security header missing from request")
	}
	for {
		s.mu.Lock()
		resp, notify := s.response(), s.notify
		s.mu.Unlock()
		if len(resp.Svids) == 0 {
			// The real agent returns PermissionDenied when no entry matches the caller.
			return status.Error(codes.PermissionDenied, "no identity issued")
		}
		if err := stream.Send(resp); err != nil {
			return err
		}
		select {
		case <-notify:
		case <-stream.Context().Done():
			return nil
		}
	}
}

func (s *Server) response() *workload.X509SVIDResponse {
	resp := &workload.X509SVIDResponse{}
	bundle := concatDER(s.ca.Bundle().X509Authorities())
	for _, sv := range s.svids {
		keyDER, _ := x509.MarshalPKCS8PrivateKey(sv.PrivateKey)
		resp.Svids = append(resp.Svids, &workload.X509SVID{
			SpiffeId:    sv.ID.String(),
			X509Svid:    concatDER(sv.Certificates),
			X509SvidKey: keyDER,
			Bundle:      bundle,
		})
	}
	return resp
}
```

`X509Source` also calls `FetchX509Bundles`. Check whether that's needed for the version you
vendored; if so, implement it the same way, sending `{"<td>": bundle}`.

## Task 3: a test that uses it

**New file:** `pkg/testing/fakespire/server_test.go`

```go
func TestFakeWorkloadAPI(t *testing.T) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	ca := NewCA(t, "ocm-hub.example.org")
	srv := Start(t, ca)

	first := ca.Issue(t, "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work", time.Hour)
	srv.SetSVIDs(first)

	src, err := workloadapi.NewX509Source(ctx, workloadapi.WithClientOptions(workloadapi.WithAddr(srv.Addr)))
	require.NoError(t, err)
	defer src.Close()

	got, err := src.GetX509SVID()
	require.NoError(t, err)
	require.Equal(t, first.ID, got.ID)
	require.Equal(t, first.Certificates[0].SerialNumber, got.Certificates[0].SerialNumber)

	// Rotate.
	second := ca.Issue(t, "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work", time.Hour)
	srv.SetSVIDs(second)
	require.Eventually(t, func() bool {
		s, _ := src.GetX509SVID()
		return s.Certificates[0].SerialNumber.Cmp(second.Certificates[0].SerialNumber) == 0
	}, 5*time.Second, 50*time.Millisecond)
}
```

| Input | Expected output |
| --- | --- |
| One SVID for `.../agent/work` | `GetX509SVID()` returns it |
| `SetSVIDs` with a new SVID | `GetX509SVID()` returns the new one within 5s |
| `SetSVIDs` with two SVIDs | stream sends both (used on Day 7) |
| No SVIDs | `NewX509Source` fails with `PermissionDenied` |

**Run:**

```shell
dgo 'go test ./pkg/testing/fakespire/ -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

**Output:**

```text
--- PASS: TestFakeWorkloadAPI (0.08s)
--- PASS: TestFakeWorkloadAPINoSVID (0.01s)
ok  	open-cluster-management.io/sdk-go/pkg/testing/fakespire	0.112s
```

## Task 4: commit

```shell
git add -A && git commit -s -m ":seedling: add a fake SPIRE Workload API for tests"
```

## End of day

| Output | Used by |
| --- | --- |
| `fakespire.NewCA`, `CA.Issue`, `CA.Bundle` | Days 7 to 9, 15, 17, 22 |
| `fakespire.Start`, `Server.SetSVIDs`, `Server.Addr` | same |

## Rules check

- [ ] Test-only: the package is under `pkg/testing` and no production code imports it.

## Result (2026-11-13)

sdk-go `3ed6eb13`, pushed. go-spiffe's `X509Source` only calls `FetchX509SVID`, so
`FetchX509Bundles` isn't implemented. Added `Server.Federate(cas...)` (federated bundles in the
response, keyed `spiffe://<td>`) and `Server.Stop()`. The socket lives in a short
`os.MkdirTemp` directory: unix socket paths are limited to about 100 bytes.
