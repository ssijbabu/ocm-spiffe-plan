# Day 2: API, ClusterManager driver and feature gate

**Status:** done. api commits `aaceeb2d` (driver and feature gate), `c77b5174` (agent trust
domains), `4d9027ee` (comments), on `fork/spire`.

**Goal:** the `ClusterManager` accepts `authType: spiffe` with its settings, including the trust
domains it accepts agents from; the hub can expose a `grpc-spiffe` endpoint; and the
`SPIFFERegistration` feature gate exists.

**Repo:** `api`, branch `spire`.

## Before you start

| Input | Where |
| --- | --- |
| Driver types | `operator/v1/types_clustermanager.go`: `RegistrationDriverHub`, `EndpointExposure`, auth type constants |
| Feature gates | `feature/feature.go` |
| Design | proposal, API ("Hub"); for several trust domains, `spire-impl-by-mode`, "Hub setup" |
| Existing patterns | durations: `AppliedManifestWorkEvictionGracePeriod` in `types_klusterlet.go` (`*metav1.Duration`, `Type=string`, pattern) |

The code before Day 2:

```go
const (
	AwsIrsaAuthType = "awsirsa"
	CSRAuthType     = "csr"
	GRPCAuthType    = "grpc"
)

type RegistrationDriverHub struct {
	// +kubebuilder:validation:Enum=csr;awsirsa;grpc
	AuthType string                  `json:"authType,omitempty"`
	CSR      *CSRConfig              `json:"csr,omitempty"`
	AwsIrsa  *AwsIrsaConfig          `json:"awsirsa,omitempty"`
	GRPC     *GRPCRegistrationConfig `json:"grpc,omitempty"`
}

type EndpointExposure struct {
	// +kubebuilder:validation:Enum=grpc;https
	Protocol string `json:"protocol"`
	GRPC  *Endpoint `json:"grpc,omitempty"`
	HTTPS *Endpoint `json:"https,omitempty"`
}
```

## Task 1: add the feature gate

**Input:** `feature/feature.go`.

**Change:**

```go
	// SPIFFERegistration enables the spiffe registration driver. Agents and add-on agents
	// authenticate to the hub gRPC server only with SPIRE-issued X.509-SVIDs, for registration,
	// renewal and revocation. OCM issues no certificate or token in this mode.
	SPIFFERegistration featuregate.Feature = "SPIFFERegistration"
```

Added, `{Default: false, PreRelease: featuregate.Alpha}`, to:

- `DefaultHubRegistrationFeatureGates`, used by the hub controllers and gRPC server;
- `DefaultSpokeRegistrationFeatureGates`, used by the registration agent;
- `DefaultSpokeWorkFeatureGates`, used by the work agent;
- `DefaultHubAddonManagerFeatureGates`, used by the add-on manager.

**Output:** `go build ./...` passes. The gate is off by default everywhere.

## Task 2: add the auth type and the spiffe block

**Input:** `operator/v1/types_clustermanager.go`.

**Change (as committed):**

```go
const (
	// ... existing ...
	// SPIFFEAuthType represents the authentication type that uses SPIRE-issued SPIFFE
	// X.509-SVIDs over gRPC. OCM issues no certificate or token in this mode.
	SPIFFEAuthType = "spiffe"
)

const (
	// GRPCSPIFFEProtocol is the endpoint exposure protocol of the separate gRPC listener used
	// by the spiffe registration driver.
	GRPCSPIFFEProtocol = "grpc-spiffe"
)

// +kubebuilder:validation:XValidation:rule="self.authType != 'spiffe' || has(self.spiffe)",message="spiffe is required when authType is spiffe"
type RegistrationDriverHub struct {
	// authType is the type of the authentication used by hub to initialize the Hub cluster.
	// Possible values are csr, awsirsa, grpc and spiffe.
	// +required
	// +kubebuilder:default:=csr
	// +kubebuilder:validation:Enum=csr;awsirsa;grpc;spiffe
	AuthType string `json:"authType,omitempty"`
	// ... CSR, AwsIrsa, GRPC unchanged ...

	// spiffe represents the configuration for the spiffe driver. It is required when
	// authType is spiffe. Effective only when the SPIFFERegistration feature gate is enabled.
	// +optional
	SPIFFE *SPIFFERegistrationConfig `json:"spiffe,omitempty"`
}

// SPIFFERegistrationConfig configures the spiffe registration driver on the hub. Agents
// authenticate with SPIRE-issued X.509-SVIDs; nothing here is secret.
type SPIFFERegistrationConfig struct {
	// trustDomain is the hub's own SPIFFE trust domain, for example ocm-hub.example.org. The
	// hub gRPC server's SVID is in it. Agents may be in other trust domains; see
	// agentTrustDomains.
	// +required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=255
	// +kubebuilder:validation:Pattern=`^[a-z0-9._-]+$`
	TrustDomain string `json:"trustDomain"`

	// agentTrustDomains are the trust domains the hub accepts Klusterlet and add-on agents
	// from. An agent whose SVID comes from any other trust domain is refused. Each managed
	// cluster is pinned to one of them with the open-cluster-management.io/spiffe-trust-domain
	// annotation. The hub's SPIRE must federate with every trust domain listed here other than
	// its own. If it is empty, only trustDomain is accepted.
	// +optional
	// +listType=set
	// +kubebuilder:validation:MaxItems=64
	// +kubebuilder:validation:items:MinLength=1
	// +kubebuilder:validation:items:MaxLength=255
	// +kubebuilder:validation:items:Pattern=`^[a-z0-9._-]+$`
	AgentTrustDomains []string `json:"agentTrustDomains,omitempty"`

	// serverSPIFFEID is the SPIFFE ID the hub gRPC server presents to agents. It must be in
	// trustDomain; the cluster-manager operator reports a Degraded condition otherwise.
	// If it is not set, spiffe://<trustDomain>/ocm/hub/grpc-server is used.
	// +optional
	// +kubebuilder:validation:MaxLength=512
	// +kubebuilder:validation:Pattern=`^spiffe://[a-z0-9._-]+(/[A-Za-z0-9._-]+)+$`
	ServerSPIFFEID string `json:"serverSPIFFEID,omitempty"`

	// workloadAPISocket is the address of the local SPIRE agent's Workload API, as mounted
	// into the gRPC server pod by the SPIFFE CSI driver.
	// +optional
	// +kubebuilder:default:="unix:///spiffe-workload-api/spire-agent.sock"
	WorkloadAPISocket string `json:"workloadAPISocket,omitempty"`

	// maxSVIDLifetime must be at least the longest SVID TTL SPIRE issues to OCM agents, in any
	// of agentTrustDomains. When a ManagedCluster is deleted, the hub keeps it terminating for
	// this long plus maxClockSkew, so no SVID issued before the deletion can recreate it.
	// +optional
	// +kubebuilder:default:="24h"
	// +kubebuilder:validation:Type=string
	// +kubebuilder:validation:Pattern="^([0-9]+(s|m|h))+$"
	MaxSVIDLifetime *metav1.Duration `json:"maxSVIDLifetime,omitempty"`

	// maxClockSkew is the allowed clock difference between the hub and the SPIRE servers of
	// every trust domain. It is used by the SVID validity check and added to the delete
	// guard's wait.
	// +optional
	// +kubebuilder:default:="5m"
	// +kubebuilder:validation:Type=string
	// +kubebuilder:validation:Pattern="^([0-9]+(s|m|h))+$"
	MaxClockSkew *metav1.Duration `json:"maxClockSkew,omitempty"`
}
```

Field summary:

| Field | Default | Purpose |
| --- | --- | --- |
| `trustDomain` | none (required) | The hub's own trust domain; the hub gRPC server's SVID is in it. |
| `agentTrustDomains` | empty, meaning `[<trustDomain>]` | Trust domains whose Klusterlet and add-on agents the hub accepts. Any other trust domain is refused. Each cluster is pinned to one of them. |
| `serverSPIFFEID` | `spiffe://<trustDomain>/ocm/hub/grpc-server` (filled by the operator) | The SVID the hub gRPC server presents. |
| `workloadAPISocket` | `unix:///spiffe-workload-api/spire-agent.sock` | The local SPIRE agent's Workload API. |
| `maxSVIDLifetime` | `24h` | At least the longest SVID TTL in any agent trust domain; the delete guard waits this long. |
| `maxClockSkew` | `5m` | Allowed clock difference between the hub and every trust domain's SPIRE. |

Notes from implementing:

- **Validation the CRD can't do.** The first version had a CEL rule checking that
  `serverSPIFFEID` is in `trustDomain`. The CRD then fails to install with
  `estimated rule cost exceeds budget by factor of 5.0x`: `registrationDrivers` has no `maxItems`,
  so every string rule on its items is multiplied by a huge estimated list size. Lowering
  `maxLength` doesn't help. Adding `maxItems` would change an existing upstream field, so the rule
  was dropped. The pattern still makes sure `serverSPIFFEID` is a SPIFFE ID, and the
  cluster-manager operator checks the trust domain (Day 14; `questions.md` Q4). The cheap rule on
  `RegistrationDriverHub` (`has(self.spiffe)`) costs nothing and stays.
- **The `agentTrustDomains` default.** A CRD default can't copy another field, so it stays empty
  in the stored object, and the operator treats empty as `[trustDomain]` (Day 14).
- **Durations** follow the repo's existing pattern: `*metav1.Duration` with `Type=string` and a
  pattern. The stored value is the string you wrote (`24h`), and Go reads it as `24h0m0s`.
- **Trust domain pattern:** lowercase letters, digits, `.`, `-` and `_`, as in the SPIFFE spec.
  `listType=set` makes the API server refuse duplicates in `agentTrustDomains`.

## Task 3: allow `grpc-spiffe` endpoint exposure

**Input:** `EndpointExposure` in the same file.

**Change (as committed):**

```go
	// protocol is the protocol used for the endpoint, could be https, grpc or grpc-spiffe.
	// grpc-spiffe is the separate gRPC listener of the spiffe registration driver; its
	// exposure is configured in the grpc field.
	// +kubebuilder:default:=grpc
	// +kubebuilder:validation:Enum=grpc;https;grpc-spiffe
	// +required
	Protocol string `json:"protocol"`

	// grpc represents the configuration for grpc endpoint. It is also used for the
	// grpc-spiffe endpoint.
	GRPC *Endpoint `json:"grpc,omitempty"`
```

No new field: `grpc-spiffe` uses the `grpc` field for its exposure type (hostname, loadBalancer or
route).

## Task 4: regenerate

**Steps:**

```shell
cd /home/ssijbabu/Documents/repos/ocm/api
dgo 'make update'
git status --short
```

**Output:**

```text
 M feature/feature.go
 M operator/v1/0000_01_operator.open-cluster-management.io_clustermanagers.crd.yaml
 M operator/v1/types_clustermanager.go
 M operator/v1/zz_generated.deepcopy.go
 M operator/v1/zz_generated.model_name.go
```

Check the CRD has the new schema:

```shell
grep -n 'grpc-spiffe\|- spiffe$\|agentTrustDomains\|spiffe is required\|default: 24h\|default: 5m' \
  operator/v1/0000_01_operator.open-cluster-management.io_clustermanagers.crd.yaml
```

**Output (line numbers vary):**

```text
511:                          - spiffe
...:                              agentTrustDomains:
585:                              default: 5m
592:                              default: 24h
631:                      - message: spiffe is required when authType is spiffe
899:                          - grpc-spiffe
```

## Task 5: integration tests for the ClusterManager

**File:** `test/integration/api/clustermanager_spiffe_test.go` (new; Ginkgo v1 with dot imports,
like `clustermanager_test.go`).

```go
var _ = Describe("ClusterManager API test with the spiffe registration driver", func() {
	var clusterManagerName string

	BeforeEach(func() {
		clusterManagerName = fmt.Sprintf("cm-%s", rand.String(5))
	})

	newClusterManager := func(driver operatorv1.RegistrationDriverHub) *operatorv1.ClusterManager {
		return &operatorv1.ClusterManager{
			ObjectMeta: metav1.ObjectMeta{Name: clusterManagerName},
			Spec: operatorv1.ClusterManagerSpec{
				RegistrationConfiguration: &operatorv1.RegistrationHubConfiguration{
					RegistrationDrivers: []operatorv1.RegistrationDriverHub{driver},
				},
			},
		}
	}

	It("accepts agents from several trust domains", func() {
		cm, err := create(newClusterManager(operatorv1.RegistrationDriverHub{
			AuthType: operatorv1.SPIFFEAuthType,
			SPIFFE: &operatorv1.SPIFFERegistrationConfig{
				TrustDomain:       "ocm-hub.example.org",
				AgentTrustDomains: []string{"ocm-hub.example.org", "east.example.org", "west.example.org"},
			},
		}))
		Expect(err).ToNot(HaveOccurred())
		Expect(cm.Spec.RegistrationConfiguration.RegistrationDrivers[0].SPIFFE.AgentTrustDomains).Should(
			Equal([]string{"ocm-hub.example.org", "east.example.org", "west.example.org"}))
	})
	// ... the other cases below ...
})
```

| Spec | Input | Expected output |
| --- | --- | --- |
| minimal driver | `trustDomain: ocm-hub.example.org` only | created; `serverSPIFFEID` empty; socket, `24h`, `5m` defaulted |
| all settings | every field set | stored as given |
| several trust domains | `agentTrustDomains: [ocm-hub, east, west]` | stored in order |
| no agent trust domains | not set | stays empty (operator uses `trustDomain`) |
| invalid agent trust domain | `[east.example.org, West_Example]` | refused |
| repeated agent trust domain | `[east.example.org, east.example.org]` | refused (`listType=set`) |
| missing spiffe block | `authType: spiffe`, no `spiffe` | refused: `spiffe is required when authType is spiffe` |
| invalid trust domain | `OCM_Hub` | refused |
| server ID not a SPIFFE ID | `https://ocm-hub.example.org/...` | refused |
| invalid duration | `maxSVIDLifetime: 1.5s` | refused by the pattern |
| spiffe next to grpc | two drivers | created |
| `grpc-spiffe` exposure | `protocol: grpc-spiffe`, `grpc.type: loadBalancer` | created |
| unknown protocol | `protocol: grpc-other` | refused |

**Run:**

```shell
dgo 'make test-api-integration' 2>&1 | tail -3
```

**Output (after Days 2 and 3):**

```text
Ran 202 of 202 Specs in 5.9 seconds
SUCCESS! -- 202 Passed | 0 Failed | 0 Pending | 0 Skipped
```

The baseline had 174 specs.

## Task 6: verify and commit

`make verify` regenerates and then runs `git diff --exit-code`, so commit first. Its
golangci-lint (v2.4.0) can't read Go 1.27 export data, so run it with Go 1.26:

```shell
git add -A && git commit -s -m ":sparkles: add spiffe registration driver to ClusterManager"
DGO_IMAGE=golang:1.26 dgo 'make verify' 2>&1 | tail -2
```

**Output:**

```text
/go/bin/golangci-lint run --timeout=3m ./...
0 issues.
```

## Example: the ClusterManager this enables

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: ClusterManager
metadata:
  name: cluster-manager
spec:
  registrationConfiguration:
    featureGates:
    - feature: SPIFFERegistration
      mode: Enable
    registrationDrivers:
    - authType: spiffe
      spiffe:
        # The hub's own trust domain. The hub's SVID is
        # spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        trustDomain: ocm-hub.example.org

        # Trust domains the hub accepts agents from. An agent whose SVID
        # comes from any other trust domain is refused.
        agentTrustDomains:
        - ocm-hub.example.org   # cluster1
        - east.example.org      # cluster2
        - west.example.org      # cluster3
  serverConfiguration:
    endpointsExposure:
    - protocol: grpc-spiffe   # the SPIFFE listener
      grpc:
        type: loadBalancer    # must pass TLS through
```

After it is applied, `kubectl get clustermanager cluster-manager -o yaml` shows
`workloadAPISocket`, `maxSVIDLifetime: 24h` and `maxClockSkew: 5m` filled in. `serverSPIFFEID`
stays empty (the operator computes it). With `agentTrustDomains` left out, only
`ocm-hub.example.org` agents are accepted.

## Rules check

- [x] No field holds a key, certificate or token.
- [x] Everything is behind `SPIFFERegistration`.
