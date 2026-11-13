# Day 8: server TLS from SPIRE and the connection registry

**Goal:** the shared gRPC server can serve with SPIFFE mTLS from the Workload API, records each
client's SPIFFE ID and SVID expiry, and closes each connection when its SVID expires.

**Repo:** `sdk-go`, branch `spire`, package `pkg/server/grpc`.

## Before you start

| Input | Where |
| --- | --- |
| `spiffe.NewX509Source` | Day 7 |
| Server builder | `pkg/server/grpc/server.go`: `NewGRPCServer(opts).WithAuthenticator(...).Run(ctx)` |

What `Run` does today: it always builds TLS from `TLSCertFile` and `TLSKeyFile` with a
`certwatcher`, and an optional `ClientCAFile`. There's no way to pass other credentials, so this is
the first thing to change.

## Task 1: let the server take transport credentials

**Change** `server.go`:

```go
type GRPCServer struct {
	// ... existing fields ...
	creds credentials.TransportCredentials // when set, replaces the certificate-file TLS setup
	extraUnary  []grpc.UnaryServerInterceptor
	extraStream []grpc.StreamServerInterceptor
}

// WithTransportCredentials makes the server use creds instead of TLSCertFile/TLSKeyFile.
func (b *GRPCServer) WithTransportCredentials(creds credentials.TransportCredentials) *GRPCServer {
	b.creds = creds
	return b
}

// WithInterceptors appends interceptors after authn and authz.
func (b *GRPCServer) WithInterceptors(u []grpc.UnaryServerInterceptor, s []grpc.StreamServerInterceptor) *GRPCServer {
	b.extraUnary, b.extraStream = append(b.extraUnary, u...), append(b.extraStream, s...)
	return b
}
```

In `Run`, skip the `certwatcher` block when `b.creds != nil` and use `grpc.Creds(b.creds)`. Also
make `GRPCServerOptions.Validate()` stop requiring `TLSCertFile`/`TLSKeyFile` in that case. Keep
the existing path byte-for-byte the same otherwise, so the `grpc` listener doesn't change.

## Task 2: the connection registry

**New file:** `pkg/server/grpc/spiffe/registry.go`

```go
package spiffe

// Registry tracks open SPIFFE connections and closes them at SVID expiry or on demand.
type Registry struct {
	mu    sync.Mutex
	conns map[*trackedConn]struct{}
	now   func() time.Time
	afterFunc func(time.Duration, func()) stopper // time.AfterFunc, replaceable in tests
}

type trackedConn struct {
	net.Conn
	id       ocmauthn.OCMIdentity
	spiffeID spiffeid.ID
	timer    stopper
	once     sync.Once
	reg      *Registry
}

func (c *trackedConn) Close() error {
	c.once.Do(func() { c.timer.Stop(); c.reg.remove(c) })
	return c.Conn.Close()
}

// track registers conn and arms a timer that closes it at notAfter.
func (r *Registry) track(conn net.Conn, id ocmauthn.OCMIdentity, sid spiffeid.ID, notAfter time.Time) *trackedConn {
	tc := &trackedConn{Conn: conn, id: id, spiffeID: sid, reg: r}
	tc.timer = r.afterFunc(notAfter.Sub(r.now()), func() {
		klog.Infof("closing connection of %s: SVID expired at %s", sid, notAfter)
		_ = tc.Close()
	})
	r.mu.Lock()
	r.conns[tc] = struct{}{}
	r.mu.Unlock()
	return tc
}

// CloseCluster closes every connection of a cluster's agents and add-on agents.
func (r *Registry) CloseCluster(cluster string) int { return r.closeWhere(func(i ocmauthn.OCMIdentity) bool { return i.ClusterName == cluster }) }

// CloseAddOn closes the connections of one add-on's agents on one cluster.
func (r *Registry) CloseAddOn(cluster, addon string) int {
	return r.closeWhere(func(i ocmauthn.OCMIdentity) bool { return i.ClusterName == cluster && i.AddOnName == addon })
}
```

`closeWhere` collects matching connections under the lock, releases the lock, then closes them,
and returns how many it closed (useful in logs and tests).

## Task 3: the credentials wrapper

**New file:** `pkg/server/grpc/spiffe/credentials.go`

```go
// NewServerCredentials returns gRPC transport credentials that do SPIFFE mTLS with the
// server's SVID, accept any client SVID in trustDomain, and track each connection.
func NewServerCredentials(src *workloadapi.X509Source, td spiffeid.TrustDomain, reg *Registry) credentials.TransportCredentials {
	tlsCfg := tlsconfig.MTLSServerConfig(src, src, tlsconfig.AuthorizeMemberOf(td))
	return &trackingCreds{TransportCredentials: credentials.NewTLS(tlsCfg), td: td, reg: reg}
}

type trackingCreds struct {
	credentials.TransportCredentials
	td  spiffeid.TrustDomain
	reg *Registry
}

func (c *trackingCreds) ServerHandshake(raw net.Conn) (net.Conn, credentials.AuthInfo, error) {
	conn, info, err := c.TransportCredentials.ServerHandshake(raw)
	if err != nil {
		return nil, nil, err
	}
	leaf := info.(credentials.TLSInfo).State.PeerCertificates[0]
	sid, err := x509svid.IDFromCert(leaf)
	if err != nil {
		conn.Close()
		return nil, nil, err
	}
	id, err := ocmauthn.ParseAgentID(sid, c.td)
	if err != nil {
		conn.Close()
		return nil, nil, err
	}
	return c.reg.track(conn, id, sid, leaf.NotAfter), info, nil
}

func (c *trackingCreds) Clone() credentials.TransportCredentials {
	return &trackingCreds{TransportCredentials: c.TransportCredentials.Clone(), td: c.td, reg: c.reg}
}
```

A client whose ID doesn't parse is dropped during the handshake, before it can send anything.

## Task 4: tests

**New file:** `pkg/server/grpc/spiffe/registry_test.go`. Use a fake clock: `afterFunc` records
the duration and the function, and the test fires it.

| Case | Input | Expected output |
| --- | --- | --- |
| Timer armed at expiry | connect at 10:40 with `NotAfter` 11:00 | `afterFunc` called with 20m |
| Expiry closes | fire the 11:00 timer | the connection is closed; registry is empty; next unary call fails with `Unavailable` |
| Open stream ends | stream open, fire the timer | the stream's `Recv` returns an error |
| Close on demand | 2 connections for cluster1, 1 for cluster2; `CloseCluster("cluster1")` | returns 2; cluster2 still works |
| Add-on close | work agent + `my-addon` agent on cluster1; `CloseAddOn("cluster1","my-addon")` | returns 1; work agent still works |
| Bad client ID | client SVID `spiffe://ocm-hub.example.org/ns/x/sa/y` | handshake fails; nothing tracked |
| Client close | client closes its connection | registry entry removed, timer stopped |

The end-to-end test of the 10:40 / 11:00 example with real time: issue an SVID with a 2-second
TTL, connect, sleep 3 seconds, check the connection was closed by the server.

**Run:**

```shell
dgo 'go test ./pkg/server/grpc/... -race -v -run "Registry|Credentials|SPIFFE"' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'go test ./pkg/server/grpc/...'   # the existing server tests must still pass
```

**Output:**

```text
--- PASS: TestRegistryArmsTimerAtExpiry (0.00s)
--- PASS: TestRegistryClosesAtExpiry (0.05s)
--- PASS: TestRegistryCloseCluster (0.09s)
--- PASS: TestRegistryCloseAddOn (0.07s)
--- PASS: TestCredentialsRejectsNonAgentID (0.04s)
--- PASS: TestSVIDExpiryRealTime (3.02s)
ok  	open-cluster-management.io/sdk-go/pkg/server/grpc/spiffe	3.4s
ok  	open-cluster-management.io/sdk-go/pkg/server/grpc	1.9s
```

Run with `-race`: the registry is used from timers and handshake goroutines at the same time.

## Task 5: commit

```shell
git add -A && git commit -s -m ":sparkles: serve SPIFFE mTLS and close connections at SVID expiry

Add WithTransportCredentials to the gRPC server, SPIFFE server credentials that
record each client's identity and SVID expiry, and a registry that closes the
connection when the SVID expires or when asked to close a cluster or add-on."
```

## End of day

| Output | Used by |
| --- | --- |
| `GRPCServer.WithTransportCredentials`, `WithInterceptors` | ocm listener (Day 10) |
| `spiffe.NewServerCredentials`, `Registry`, `CloseCluster`, `CloseAddOn` | Days 10, 12 |

## Rules check

- [ ] The server's SVID comes from the Workload API; no certificate files in `spiffe` mode.
- [ ] Connections can't outlive the SVID they were opened with.

## Update: multiple agent trust domains

Server TLS uses `tlsconfig.MTLSServerConfig(src, src, authorizeMemberOfAny(agentTrustDomains))` instead of `AuthorizeMemberOf(td)`; write `authorizeMemberOfAny` as a small `tlsconfig.Authorizer`. Even if the hub's SPIRE federates with more domains, only listed ones get in. The registry also needs `CloseTrustDomain(td)` for when a domain is removed from the list.

## Update: agent ID templates

`trackingCreds` holds the `*authn.IDScheme` instead of a trust domain and calls
`c.scheme.Parse(sid)`. Build the server TLS authorizer from `scheme.TrustDomains()`, so the list of
trust domains has one source. The registry keys connections by `OCMIdentity.ClusterName` and
`AddOnName` as before; `Namespace` is never a key.
