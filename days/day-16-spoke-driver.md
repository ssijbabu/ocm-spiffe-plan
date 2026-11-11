# Day 16: the spoke registration driver

**Goal:** a registration agent started with `--registration-auth=spiffe` joins the hub with its SVID
only: no bootstrap kubeconfig, no CSR, no hub-kubeconfig Secret.

**Repo:** `ocm`, branch `spire`, new package `pkg/registration/register/spiffe`.

## Before you start

| Input | Where |
| --- | --- |
| Driver interface | `pkg/registration/register/interface.go`: `RegisterDriver` |
| Existing gRPC driver | `pkg/registration/register/grpc/spoke_driver.go` (closest example) |
| Factory | `pkg/registration/register/factory/options.go` |
| Client options | sdk-go `NewSPIFFEGRPCOptions` (Day 7) |

The factory today:

```go
func (s *Options) Driver(secretOption register.SecretOption) (register.RegisterDriver, error) {
	switch s.RegistrationAuth {
	case operatorv1.AwsIrsaAuthType:
		return awsirsa.NewAWSIRSADriver(s.AWSIRSAOption, secretOption), nil
	case operatorv1.GRPCAuthType:
		return grpc.NewGRPCDriver(s.GRPCOption, s.CSROption, secretOption)
	default:
		return csr.NewCSRDriver(s.CSROption, secretOption)
	}
}
```

## Task 1: read the interface

**Steps:** read `RegisterDriver` and every method's callers in `pkg/registration/spoke`. Write down
which methods assume a Secret (bootstrap, `hub-kubeconfig-secret`, cert rotation). For each, decide
what `spiffe` does. Example of the expected notes:

| Method (check the real names) | `csr`/`grpc` today | `spiffe` |
| --- | --- | --- |
| bootstrap / `Process` | creates a CSR, waits for approval, writes the Secret | nothing to do: return "ready" |
| `IsHubKubeConfigValid` | checks the cert in the Secret | true when the source has the expected SVID |
| `BuildClients` | builds clients from the Secret's kubeconfig or gRPC config | builds gRPC clients from `NewSPIFFEGRPCOptions` |
| `ManagedClusterDecorator` | adds labels/annotations | none |

## Task 2: options and flags

**New file:** `pkg/registration/register/spiffe/options.go`

```go
type Option struct {
	HubEndpoint       string
	HubSPIFFEID       string
	WorkloadAPISocket string
	TrustDomain       string // derived from HubSPIFFEID when empty
}

func (o *Option) AddFlags(fs *pflag.FlagSet) {
	fs.StringVar(&o.HubEndpoint, "spiffe-hub-endpoint", "", "host:port of the hub's grpc-spiffe listener.")
	fs.StringVar(&o.HubSPIFFEID, "spiffe-hub-id", "", "The only SPIFFE ID accepted from the hub.")
	fs.StringVar(&o.WorkloadAPISocket, "spiffe-workload-api-socket", "unix:///spiffe-workload-api/spire-agent.sock", "")
}

func (o *Option) Validate() error {
	if o.HubEndpoint == "" || o.HubSPIFFEID == "" {
		return errors.New("--spiffe-hub-endpoint and --spiffe-hub-id are required with --registration-auth=spiffe")
	}
	_, err := spiffeid.FromString(o.HubSPIFFEID)
	return err
}

// AgentID is the SPIFFE ID this agent expects for itself.
func (o *Option) AgentID(cluster string, role string) string {
	return fmt.Sprintf("spiffe://%s/ocm/cluster/%s/agent/%s", o.trustDomain(), cluster, role)
}
```

## Task 3: the driver

**New file:** `pkg/registration/register/spiffe/spoke_driver.go`. Implement the interface from
Task 1. The key method builds the clients:

```go
func (d *SPIFFEDriver) BuildClients(ctx context.Context, cluster string) (*register.Clients, error) {
	opts, closer, err := grpcoptions.NewSPIFFEGRPCOptions(ctx, grpcoptions.SPIFFEConfig{
		URL:               d.opt.HubEndpoint,
		HubSPIFFEID:       d.opt.HubSPIFFEID,
		AgentSPIFFEID:     d.opt.AgentID(cluster, d.role), // "registration" or "klusterlet"
		WorkloadAPISocket: d.opt.WorkloadAPISocket,
	})
	if err != nil {
		return nil, err // includes ErrSVIDNotFound with the IDs received
	}
	d.closer = closer
	// Build the same CloudEvents-based clients the grpc driver builds, from opts.
	return buildCloudEventsClients(ctx, opts, cluster)
}
```

Reuse the grpc driver's client-building code (move the shared part to a helper rather than copying
it). The difference is only where `GRPCOptions` comes from.

## Task 4: factory and flag wiring

```go
	case operatorv1.SPIFFEAuthType:
		return spiffe.NewSPIFFEDriver(s.SPIFFEOption, secretOption), nil
```

Add `SPIFFEOption` to `Options`, its flags to `AddFlags`, and its `Validate` to `Validate`. In
`pkg/registration/spoke`, don't start the Secret-based hub-kubeconfig controller when the driver
is `spiffe` (find where it's started, and add the condition there).

## Task 5: tests

**Unit** (`pkg/registration/register/spiffe/*_test.go`):

| Case | Input | Expected |
| --- | --- | --- |
| Validate | `--registration-auth=spiffe` without `--spiffe-hub-endpoint` | error naming both flags |
| Agent ID | cluster `cluster1`, role `registration`, hub ID in `ocm-hub.example.org` | `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/registration` |
| Missing SVID | fake Workload API has only `.../agent/work` | `BuildClients` error lists the received ID |
| No Secret | run a full sync with a fake kube client | no `create` action on Secrets |

**Integration:** replace the Day 15 fake agent with the real one:

```go
func runRegistrationAgentSPIFFE(cluster, socket string) context.CancelFunc {
	opts := spoke.NewSpokeAgentOptions()
	opts.RegisterDriverOption.RegistrationAuth = "spiffe"
	opts.RegisterDriverOption.SPIFFEOption.HubEndpoint = spiffeHubAddr
	opts.RegisterDriverOption.SPIFFEOption.HubSPIFFEID = "spiffe://ocm-hub.example.org/ocm/hub/grpc-server"
	opts.RegisterDriverOption.SPIFFEOption.WorkloadAPISocket = socket
	opts.AgentOptions.SpokeClusterName = cluster
	return runAgent("spiffe-agent", opts, spokeCfg)
}
```

**Run:**

```shell
dgo 'go test ./pkg/registration/register/... -v -run SPIFFE' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'make test-registration-integration ARGS="-ginkgo.focus=spiffe"' 2>&1 | tail -2
```

**Output:** unit tests pass; the 3 SPIFFE integration specs pass with the real agent.

## Task 6: commit

```shell
git add -A && git commit -s -m ":sparkles: add the spiffe registration driver for agents"
```

## Rules check

- [ ] No bootstrap Secret, no CSR, no `hub-kubeconfig-secret`.
- [ ] The agent talks only to the hub endpoint and the local Workload API.

## Update: multiple agent trust domains

`Option.TrustDomain` comes from the Klusterlet's `spiffe.trustDomain` (flag `--spiffe-trust-domain`), defaulting to the trust domain of `--spiffe-hub-id`. `AgentID` uses it, so cluster2's registration agent expects `spiffe://east.example.org/ocm/cluster/cluster2/agent/registration`.
