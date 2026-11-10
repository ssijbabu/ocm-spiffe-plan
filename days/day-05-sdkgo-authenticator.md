# Day 5: SPIFFE authenticator and expiry interceptors

**Goal:** a `SPIFFEAuthenticator` that plugs into the existing gRPC server
(`WithAuthenticator`) and puts the agent's OCM identity, SPIFFE ID and SVID expiry into the
request context, plus interceptors that refuse an RPC once the peer's SVID has expired.

**Repo:** `sdk-go`, branch `spire`, package `pkg/server/grpc/authn`.

## Before you start

| Input | Where |
| --- | --- |
| `ParseAgentID` | Day 4, `pkg/server/grpc/authn/spiffeid.go` |
| Authenticator interface | `pkg/server/grpc/authn/interface.go` |
| How the server chains authenticators | `pkg/server/grpc/server.go`, `newAuthnUnaryInterceptor` |

The interface you implement:

```go
type Authenticator interface {
	Authenticate(ctx context.Context) (context.Context, error)
}
```

`newContextWithIdentity(ctx, user, groups)` stores user and groups under `ContextUserKey` and
`ContextGroupsKey`. The authorizer (`SARAuthorizer`) reads them from there.

## Task 1: write the authenticator

**New file:** `pkg/server/grpc/authn/spiffe.go`

```go
package authn

import (
	"context"
	"time"

	"github.com/spiffe/go-spiffe/v2/spiffeid"
	"github.com/spiffe/go-spiffe/v2/svid/x509svid"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/peer"
	"google.golang.org/grpc/status"
)

const (
	ContextSPIFFEIDKey    contextKey = "spiffe-id"
	ContextSVIDExpiryKey  contextKey = "svid-expiry"
	ContextOCMIdentityKey contextKey = "ocm-identity"
)

// SPIFFEAuthenticator authenticates a peer by the X.509-SVID it presented in the TLS handshake.
// The TLS layer has already verified the chain against the SPIRE trust bundle.
type SPIFFEAuthenticator struct {
	trustDomain spiffeid.TrustDomain
	now         func() time.Time
}

func NewSPIFFEAuthenticator(td spiffeid.TrustDomain) *SPIFFEAuthenticator {
	return &SPIFFEAuthenticator{trustDomain: td, now: time.Now}
}

func (a *SPIFFEAuthenticator) Authenticate(ctx context.Context) (context.Context, error) {
	p, ok := peer.FromContext(ctx)
	if !ok {
		return ctx, status.Error(codes.Unauthenticated, "no peer found")
	}
	tlsInfo, ok := p.AuthInfo.(credentials.TLSInfo)
	if !ok || len(tlsInfo.State.VerifiedChains) == 0 || len(tlsInfo.State.VerifiedChains[0]) == 0 {
		return ctx, status.Error(codes.Unauthenticated, "no verified peer certificate")
	}
	leaf := tlsInfo.State.VerifiedChains[0][0]
	id, err := x509svid.IDFromCert(leaf)
	if err != nil {
		return ctx, status.Errorf(codes.Unauthenticated, "peer certificate is not an X.509-SVID: %v", err)
	}
	if !a.now().Before(leaf.NotAfter) {
		return ctx, status.Errorf(codes.Unauthenticated, "SVID for %s expired at %s", id, leaf.NotAfter)
	}
	identity, err := ParseAgentID(id, a.trustDomain)
	if err != nil {
		return ctx, status.Error(codes.Unauthenticated, err.Error())
	}
	ctx = newContextWithIdentity(ctx, identity.User, identity.Groups)
	ctx = context.WithValue(ctx, ContextSPIFFEIDKey, id)
	ctx = context.WithValue(ctx, ContextSVIDExpiryKey, leaf.NotAfter)
	return context.WithValue(ctx, ContextOCMIdentityKey, identity), nil
}

// OCMIdentityFromContext returns the agent identity set by SPIFFEAuthenticator.
func OCMIdentityFromContext(ctx context.Context) (OCMIdentity, bool) {
	id, ok := ctx.Value(ContextOCMIdentityKey).(OCMIdentity)
	return id, ok
}
```

How errors are handled: `newAuthnUnaryInterceptor` tries each authenticator in order and calls
the handler on the first success; if all fail, it returns the last error. The SPIFFE listener is
built with only this authenticator (Day 10), so its error ends the request. Never add it to the
existing `grpc` listener's list: a token or OCM-signed certificate must not be accepted on the
SPIFFE listener, and an SVID must not be accepted on the `grpc` one.

## Task 2: expiry interceptors

The connection is closed at SVID expiry on Day 8. These interceptors are the second check: an RPC
that arrives on a connection whose SVID has expired fails, even if the close hasn't happened yet.

**Add to `spiffe.go`:**

```go
// SVIDExpiryUnaryInterceptor refuses a unary RPC whose peer SVID has expired.
// It must run after the authn interceptor.
func SVIDExpiryUnaryInterceptor(now func() time.Time) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, h grpc.UnaryHandler) (any, error) {
		if err := checkExpiry(ctx, now); err != nil {
			return nil, err
		}
		return h(ctx, req)
	}
}

// SVIDExpiryStreamInterceptor does the same for streams.
func SVIDExpiryStreamInterceptor(now func() time.Time) grpc.StreamServerInterceptor {
	return func(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, h grpc.StreamHandler) error {
		if err := checkExpiry(ss.Context(), now); err != nil {
			return err
		}
		return h(srv, ss)
	}
}

func checkExpiry(ctx context.Context, now func() time.Time) error {
	exp, ok := ctx.Value(ContextSVIDExpiryKey).(time.Time)
	if !ok {
		return status.Error(codes.Unauthenticated, "no SVID expiry in context")
	}
	if !now().Before(exp) {
		return status.Errorf(codes.Unauthenticated, "SVID expired at %s", exp)
	}
	return nil
}
```

A stream that is already open isn't checked again by the interceptor. The connection close on
Day 8 handles that case.

## Task 3: unit tests

**New file:** `pkg/server/grpc/authn/spiffe_test.go`. Build certificates in the test with
`crypto/x509`: a CA, and a leaf with a SPIFFE URI SAN.

```go
func newSVID(t *testing.T, id string, notAfter time.Time) *x509.Certificate {
	t.Helper()
	u, _ := url.Parse(id)
	tmpl := &x509.Certificate{
		SerialNumber: big.NewInt(1),
		NotBefore:    time.Now().Add(-time.Minute),
		NotAfter:     notAfter,
		URIs:         []*url.URL{u},
	}
	key, _ := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	der, err := x509.CreateCertificate(rand.Reader, tmpl, tmpl, &key.PublicKey, key)
	if err != nil {
		t.Fatal(err)
	}
	cert, _ := x509.ParseCertificate(der)
	return cert
}

func ctxWithPeerCert(cert *x509.Certificate) context.Context {
	return peer.NewContext(context.Background(), &peer.Peer{
		AuthInfo: credentials.TLSInfo{State: tls.ConnectionState{
			VerifiedChains: [][]*x509.Certificate{{cert}},
		}},
	})
}
```

| Case | Input | Expected output |
| --- | --- | --- |
| Valid work agent | leaf `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work`, expires in 1h | ctx user `system:open-cluster-management:cluster1:work`; `OCMIdentityFromContext` returns cluster `cluster1`, role `work` |
| Expired SVID | same ID, `NotAfter` 1 minute ago | `codes.Unauthenticated`, message contains `expired` |
| No peer | `context.Background()` | `codes.Unauthenticated`, `no peer found` |
| No verified chain | TLSInfo with empty `VerifiedChains` | `codes.Unauthenticated` |
| Certificate without SPIFFE URI | leaf with no URIs | `codes.Unauthenticated`, `not an X.509-SVID` |
| Other trust domain | `spiffe://other.example.org/...` | `codes.Unauthenticated`, `not in trust domain` |
| Interceptor, expired | ctx with `ContextSVIDExpiryKey` in the past | handler is not called; `codes.Unauthenticated` |
| Interceptor, valid | expiry in the future | handler is called once |

Use a fixed `now` (set `a.now = func() time.Time { return t0 }`) so tests don't depend on the clock.

**Run:**

```shell
dgo 'go test ./pkg/server/grpc/authn/... -v -run "SPIFFE|Expiry|ParseAgentID"' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'go vet ./pkg/server/grpc/...'
```

**Output:**

```text
--- PASS: TestParseAgentID (0.00s)
--- PASS: TestSPIFFEAuthenticator (0.01s)
--- PASS: TestSVIDExpiryInterceptors (0.00s)
ok  	open-cluster-management.io/sdk-go/pkg/server/grpc/authn	0.031s
```

## Task 4: commit

```shell
git add -A && git commit -s -m ":sparkles: add SPIFFE authenticator and SVID expiry interceptors"
```

## End of day

| Output | Check |
| --- | --- |
| `SPIFFEAuthenticator`, `OCMIdentityFromContext` | unit tests pass |
| Expiry interceptors | unit tests pass |

## Rules check

- [ ] The authenticator only reads the SVID the TLS layer verified; it doesn't load any key or
      certificate file.
- [ ] Expired SVIDs are refused.
