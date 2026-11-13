# Day 10: the SPIFFE listener in the hub gRPC server

**Goal:** the hub's gRPC server process can run a second, separate listener that uses only SPIFFE
mTLS and the SPIFFE authenticator. The existing `grpc` listener is unchanged.

**Repo:** `ocm`, branch `spire`, package `pkg/server/grpc` and `pkg/cmd/hub`.

## Before you start

| Input | Where |
| --- | --- |
| api head SHA | Day 3 |
| sdk-go head SHA | Day 9 |
| Server command | `pkg/cmd/hub/grpc_server.go` (`server grpc`) |
| Server wiring | `pkg/server/grpc/options.go`: `GRPCServerOptions.Run` |

Today's `Run` registers the CloudEvents services on one broker, then:

```go
authorizer := grpcauthz.NewSARAuthorizer(clients.KubeClient)
return sdkgrpc.NewGRPCServer(serverOptions).
	WithAuthenticator(grpcauthn.NewTokenAuthenticator(clients.KubeClient)).
	WithAuthenticator(grpcauthn.NewMtlsAuthenticator()).
	WithUnaryAuthorizer(authorizer).
	WithStreamAuthorizer(authorizer).
	WithRegisterFunc(func(s *grpc.Server) { pbv1.RegisterCloudEventServiceServer(s, grpcEventServer) }).
	WithExtraMetrics(cemetrics.CloudEventsGRPCMetrics()...).
	Run(ctx)
```

## Task 1: point ocm at the forks

```shell
cd /home/ssijbabu/Documents/repos/ocm/ocm
dgo 'go mod edit \
  -replace open-cluster-management.io/api=github.com/ssijbabu/api@<api-sha> \
  -replace open-cluster-management.io/sdk-go=github.com/ssijbabu/sdk-go@<sdk-go-sha> \
  && go mod tidy && go mod vendor && go build ./...'
tail -4 go.mod
```

**Output:**

```text
replace (
	open-cluster-management.io/api => github.com/ssijbabu/api v1.4.1-0.20261112...-4f2a9c1e...
	open-cluster-management.io/sdk-go => github.com/ssijbabu/sdk-go v1.4.1-0.20261113...-8be1...
)
```

Go turns the SHA into a pseudo-version. Commit this on its own (`:seedling: point api and sdk-go
at the spiffe forks`), so it's easy to drop later.

## Task 2: flags for the SPIFFE listener

**Change** `pkg/server/grpc/options.go`:

```go
type GRPCServerOptions struct {
	// ... existing fields ...
	SPIFFE SPIFFEListenerOptions
}

type SPIFFEListenerOptions struct {
	Enabled           bool
	ListenAddress     string // e.g. ":8091"
	TrustDomain       string
	ServerSPIFFEID    string
	WorkloadAPISocket string
}

func (o *GRPCServerOptions) AddFlags(fs *pflag.FlagSet) {
	// ... existing ...
	fs.BoolVar(&o.SPIFFE.Enabled, "spiffe-enabled", false, "Run the SPIFFE listener.")
	fs.StringVar(&o.SPIFFE.ListenAddress, "spiffe-listen-address", ":8091", "Address of the SPIFFE listener.")
	fs.StringVar(&o.SPIFFE.TrustDomain, "spiffe-trust-domain", "", "SPIFFE trust domain of the hub and agents.")
	fs.StringVar(&o.SPIFFE.ServerSPIFFEID, "spiffe-server-id", "", "SPIFFE ID the server presents. Default spiffe://<td>/ocm/hub/grpc-server.")
	fs.StringVar(&o.SPIFFE.WorkloadAPISocket, "spiffe-workload-api-socket", "unix:///spiffe-workload-api/spire-agent.sock", "Workload API address.")
}
```

Also add a flag that turns off the existing `grpc` listener (`--grpc-enabled`, default true), for
the case where `spiffe` is the only driver. Pick the port with the operator in mind (Day 14); check
which ports the gRPC server Deployment already uses (`8090` for grpc, `8443` for health) and avoid
them.

## Task 3: start both listeners

**Change** `Run`. Register the services once on the same broker, then build one server per
listener:

```go
	authorizer := grpcauthz.NewSARAuthorizer(clients.KubeClient)
	register := func(s *grpc.Server) { pbv1.RegisterCloudEventServiceServer(s, grpcEventServer) }

	g, ctx := errgroup.WithContext(ctx)
	if o.GRPCEnabled {
		g.Go(func() error {
			return sdkgrpc.NewGRPCServer(serverOptions).
				WithAuthenticator(grpcauthn.NewTokenAuthenticator(clients.KubeClient)).
				WithAuthenticator(grpcauthn.NewMtlsAuthenticator()).
				WithUnaryAuthorizer(authorizer).WithStreamAuthorizer(authorizer).
				WithRegisterFunc(register).
				WithExtraMetrics(cemetrics.CloudEventsGRPCMetrics()...).
				Run(ctx)
		})
	}
	if o.SPIFFE.Enabled {
		g.Go(func() error { return o.runSPIFFEListener(ctx, clients, authorizer, register) })
	}
	return g.Wait()
```

```go
func (o *GRPCServerOptions) runSPIFFEListener(ctx context.Context, clients *Clients,
	authorizer *grpcauthz.SARAuthorizer, register func(*grpc.Server)) error {
	td, err := spiffeid.TrustDomainFromString(o.SPIFFE.TrustDomain)
	if err != nil {
		return err
	}
	serverID := o.SPIFFE.ServerSPIFFEID
	if serverID == "" {
		serverID = "spiffe://" + td.Name() + "/ocm/hub/grpc-server"
	}
	src, err := ocmspiffe.NewX509Source(ctx, o.SPIFFE.WorkloadAPISocket, spiffeid.RequireFromString(serverID))
	if err != nil {
		return fmt.Errorf("SPIFFE listener: %w", err)
	}
	defer src.Close()

	o.registry = spiffeserver.NewRegistry() // used by revocation on Day 12
	opts := sdkgrpc.NewGRPCServerOptions()
	opts.ServerBindPort = portOf(o.SPIFFE.ListenAddress) // check the real field name
	return sdkgrpc.NewGRPCServer(opts).
		WithTransportCredentials(spiffeserver.NewServerCredentials(src, td, o.registry)).
		WithAuthenticator(grpcauthn.NewSPIFFEAuthenticator(td)). // the only authenticator here
		WithUnaryAuthorizer(authorizer).WithStreamAuthorizer(authorizer).
		WithInterceptors(
			[]grpc.UnaryServerInterceptor{grpcauthn.SVIDExpiryUnaryInterceptor(time.Now)},
			[]grpc.StreamServerInterceptor{grpcauthn.SVIDExpiryStreamInterceptor(time.Now)}).
		WithRegisterFunc(register).
		Run(ctx)
}
```

The SPIFFE listener doesn't offer CSR or token requests. Because both listeners share one broker,
remove those two services from the SPIFFE path with an authorizer check: refuse
`CSREventDataType` and `TokenRequestDataType` events when the context has an `OCMIdentity`.
That's simpler than two brokers, and it's tested on Day 11.

## Task 4: tests

**File:** `pkg/server/grpc/options_test.go` (add) and a new `spiffe_listener_test.go`.

| Case | Input | Expected output |
| --- | --- | --- |
| Flags | `--spiffe-enabled --spiffe-trust-domain ocm-hub.example.org` | options parsed; default server ID `spiffe://ocm-hub.example.org/ocm/hub/grpc-server` |
| Bad trust domain | `--spiffe-trust-domain OCM_HUB` | `Run` returns an error naming the flag |
| No server SVID | fake Workload API returns only another ID | `Run` returns `ErrSVIDNotFound` naming the expected ID |
| SPIFFE only | `--grpc-enabled=false --spiffe-enabled` | only the SPIFFE port listens; a token client can't connect |
| Both | both enabled | a token client works on `grpc`; a SPIFFE client works on the SPIFFE port; a token client is refused on the SPIFFE port at TLS |

Run the server in-process with `fakespire` and an envtest kube-apiserver (the existing
`TestGRPCServerOptionsRunWithValidConfigFile` shows how to start it).

**Run:**

```shell
dgo 'go build ./... && go test ./pkg/server/grpc/... ./pkg/cmd/hub/... -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

**Output:**

```text
--- PASS: TestSPIFFEListenerFlags (0.00s)
--- PASS: TestSPIFFEListenerNoServerSVID (0.31s)
--- PASS: TestSPIFFEOnly (2.10s)
--- PASS: TestSPIFFEAndGRPCTogether (2.84s)
ok  	open-cluster-management.io/ocm/pkg/server/grpc	6.1s
ok  	open-cluster-management.io/ocm/pkg/cmd/hub	0.4s
```

## Task 5: commit

```shell
git add -A && git commit -s -m ":sparkles: add the SPIFFE listener to the hub gRPC server

Run a separate listener with SPIFFE mTLS from the Workload API and the SPIFFE
authenticator only, next to the existing grpc listener. Both share the
CloudEvents services and authorizer."
```

## End of day

| Output | Check |
| --- | --- |
| `replace` commit | `go build ./...` passes |
| SPIFFE listener behind flags | tests pass; existing server tests unchanged |

## Rules check

- [ ] SPIFFE listener: no token authenticator, no OCM-signed certificate, no CSR or token service.
- [ ] The server's SVID comes only from the local Workload API.

## Update: multiple agent trust domains

Add the flag `--spiffe-agent-trust-domains` (comma separated; default: the hub's trust domain), passed to the authenticator and server TLS.

Always get the accepted trust domains from `SPIFFERegistrationConfig.GetAgentTrustDomains()` (api `1fa143c5`). It adds the hub's own trust domain whether or not the admin listed it, so never read `AgentTrustDomains` directly.

## Update: agent ID templates

Two more flags, defaulting to the api constants:

```go
	fs.StringVar(&o.SPIFFE.AgentIDTemplate, "spiffe-agent-id-template", v1.DefaultSPIFFEAgentIDTemplate,
		"SPIFFE ID layout of Klusterlet agents.")
	fs.StringVar(&o.SPIFFE.AddOnAgentIDTemplate, "spiffe-addon-agent-id-template", v1.DefaultSPIFFEAddOnAgentIDTemplate,
		"SPIFFE ID layout of add-on agents.")
```

In `runSPIFFEListener`, build the scheme once and refuse to start on any error, before opening the
Workload API source:

```go
	scheme, err := grpcauthn.NewIDScheme(o.SPIFFE.AgentIDTemplate, o.SPIFFE.AddOnAgentIDTemplate,
		o.SPIFFE.AgentTrustDomains)
	if err != nil {
		return fmt.Errorf("SPIFFE listener: %w", err)
	}
	if err := scheme.CheckServerID(spiffeid.RequireFromString(serverID)); err != nil {
		return fmt.Errorf("SPIFFE listener: %w", err)
	}
	...
		WithTransportCredentials(spiffeserver.NewServerCredentials(src, scheme, o.registry)).
		WithAuthenticator(grpcauthn.NewSPIFFEAuthenticator(scheme)).
```

Test rows:

| Case | Flags | Expected |
| --- | --- | --- |
| Defaults | no template flags | an agent with `.../ocm/cluster/cluster1/agent/work` connects |
| Custom template | `--spiffe-agent-id-template=spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}` | `.../k8s/cluster1/ns/open-cluster-management-agent/ocm/work` connects; `.../ocm/cluster/cluster1/agent/work` is refused |
| Bad template | `--spiffe-agent-id-template=spiffe://{trustDomain}/{cluster}/{role}` | `Run` returns `needs at least one fixed path segment` |
| Server ID matches a template | `--spiffe-server-id=spiffe://ocm-hub.example.org/ocm/cluster/hub/agent/work` | `Run` returns `agents could present it` |

## Update: what Days 4–9 built (2026-11-13)

- **Task 1 SHAs:** api `b805cf6c`, sdk-go `7541964e`. ocm is on grpc 1.84.0, so sdk-go's move to
  grpc 1.79.3 (go-spiffe v2.9.0, `questions.md` Q10) changes nothing there.
- **Signatures:** `spiffe.NewX509Source(ctx, addr, want)` returns `*spiffe.X509Source` (with
  `Ready()` for the readiness check); `spiffeserver.NewServerCredentials(src, scheme, reg)` takes
  the `*authn.IDScheme`; `authn.NewSPIFFEAuthenticator(scheme)`.
- **Always pair** `NewServerCredentials` with `NewSPIFFEAuthenticator`: the authenticator accepts
  only the `authn.SPIFFEAuthInfo` those credentials produce (Day 5 result). Any other credentials
  make every request `Unauthenticated`.
- With `WithTransportCredentials`, the server's TLS version and cipher options don't apply; the
  SPIFFE credentials own the TLS config.
- Use `server_spiffe_test.go` in sdk-go as the model for the listener test: it runs the server
  builder end to end with `fakespire`.

## Status (2026-11-13)

Done, committed locally. See "Progress (Days 10-23)" in `questions.md` for the commit and findings.
