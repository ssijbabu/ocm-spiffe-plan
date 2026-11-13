# Day 2: API, ClusterManager driver and feature gate

**Status:** done. "Update: agent ID templates" is api `b805cf6c` (2026-11-13), pushed to `fork/spire`. api commits `aaceeb2d` (driver and feature gate), `c77b5174` (agent trust
domains), `4d9027ee` (comments), `1fa143c5` (hub's trust domain always accepted, full rule
coverage), `dddccc02` (helper test), on `fork/spire`.

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
| `agentTrustDomains` | empty | Trust domains whose Klusterlet and add-on agents the hub accepts. The hub's own `trustDomain` is always accepted too, whether or not it is listed (`GetAgentTrustDomains`, Task 2b). Any other trust domain is refused. Each cluster is pinned to one of them. |
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
- **The hub's trust domain in `agentTrustDomains`.** A CRD default can't copy another field, and
  the ClusterManager has no defaulting webhook, so the stored list is never rewritten. Instead,
  `GetAgentTrustDomains` (Task 2b) always adds the hub's trust domain, and every consumer uses it.
- **Durations** follow the repo's existing pattern: `*metav1.Duration` with `Type=string` and a
  pattern. The stored value is the string you wrote (`24h`), and Go reads it as `24h0m0s`.
- **Trust domain pattern:** lowercase letters, digits, `.`, `-` and `_`, as in the SPIFFE spec.
  `listType=set` makes the API server refuse duplicates in `agentTrustDomains`.

## Task 2b: always accept the hub's own trust domain

**Rule:** the admin may list the hub's own `trustDomain` in `agentTrustDomains` or leave it out.
Either way the hub accepts agents from it.

**Change (as committed, api `1fa143c5`)** in `operator/v1/funcs_clustermanager.go`:

```go
// GetAgentTrustDomains returns the trust domains the hub accepts agents from. The hub's own
// trustDomain is always included and comes first, whether or not agentTrustDomains lists it;
// the listed trust domains follow in their order, without duplicates or empty entries.
// It returns nil for a nil config or an empty trustDomain.
func (c *SPIFFERegistrationConfig) GetAgentTrustDomains() []string {
	if c == nil || c.TrustDomain == "" {
		return nil
	}
	domains := make([]string, 0, len(c.AgentTrustDomains)+1)
	seen := make(map[string]bool, len(c.AgentTrustDomains)+1)
	for _, td := range append([]string{c.TrustDomain}, c.AgentTrustDomains...) {
		if td == "" || seen[td] {
			continue
		}
		seen[td] = true
		domains = append(domains, td)
	}
	return domains
}
```

| `trustDomain` | `agentTrustDomains` (stored) | `GetAgentTrustDomains()` |
| --- | --- | --- |
| `ocm-hub.example.org` | not set | `[ocm-hub.example.org]` |
| `ocm-hub.example.org` | `[ocm-hub.example.org, east, west]` | `[ocm-hub.example.org, east, west]` |
| `ocm-hub.example.org` | `[east, west]` | `[ocm-hub.example.org, east, west]` (added) |
| `ocm-hub.example.org` | `[east, ocm-hub.example.org]` | `[ocm-hub.example.org, east]` (moved first, once) |
| empty | anything | `nil` |
| nil config | | `nil` |

The stored spec stays as the admin wrote it; the gRPC server (Day 10) and the operator (Day 14)
call this function and never read `AgentTrustDomains` directly.

**Unit tests:** `operator/v1/funcs_clustermanager_test.go`, one case per row above, plus a check
that calling it doesn't change the spec. Coverage of the hand-written functions:

```shell
dgo 'go test ./operator/v1/ -coverprofile=/tmp/c.out >/dev/null && go tool cover -func=/tmp/c.out | grep funcs_'
```

```text
open-cluster-management.io/api/operator/v1/funcs_clustermanager.go:4:   GetResourceRequirement  100.0%
open-cluster-management.io/api/operator/v1/funcs_clustermanager.go:15:  GetAgentTrustDomains    100.0%
open-cluster-management.io/api/operator/v1/funcs_klusterlet.go:4:       GetResourceRequirement  100.0%
```

The package total is about 1.5%: the rest of `operator/v1` is generated deepcopy code, which the
repo doesn't unit-test.

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


**Every validation rule has a test.** For each rule on the new ClusterManager fields:

| Field | Rule | Accepted case | Refused case |
| --- | --- | --- | --- |
| `authType` | enum adds `spiffe` | minimal driver | unknown protocol (enum on `protocol`) |
| `RegistrationDriverHub` | CEL `has(self.spiffe)` | minimal driver | missing spiffe block |
| `trustDomain` | required, minLength 1 | minimal driver | `trustDomain: ""` |
| `trustDomain` | maxLength 255 | 255 characters | 256 characters |
| `trustDomain` | pattern | `ocm-hub.example.org` | `OCM_Hub` |
| `agentTrustDomains` | optional | not set | |
| `agentTrustDomains` | `listType=set` | three distinct | `east` twice |
| `agentTrustDomains` | maxItems 64 | 64 entries | 65 entries |
| `agentTrustDomains` items | minLength 1 | | empty entry |
| `agentTrustDomains` items | maxLength 255 | | 256 characters |
| `agentTrustDomains` items | pattern | `east.example.org` | `West_Example` |
| hub's own trust domain | always accepted | listed, left out, not set (checked with `GetAgentTrustDomains`) | |
| `serverSPIFFEID` | pattern | `spiffe://ocm-hub.example.org/ocm/hub/grpc-server` | `https://...`, `spiffe://ocm-hub.example.org` (no path) |
| `serverSPIFFEID` | maxLength 512 | | over 512 |
| `workloadAPISocket` | default | defaulted | |
| `maxSVIDLifetime` | default `24h`, pattern | defaulted; `1h` | `1.5s` |
| `maxClockSkew` | default `5m`, pattern | defaulted; `1m` | `500ms` |
| `protocol` | enum adds `grpc-spiffe` | `grpc-spiffe` | `grpc-other` |

**Run:**

```shell
dgo 'make test-api-integration' 2>&1 | tail -3
```

**Output (after Days 2 and 3):**

```text
Ran 223 of 223 Specs in 5.6 seconds
SUCCESS! -- 223 Passed | 0 Failed | 0 Pending | 0 Skipped
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

## Update: agent ID templates (done: api `b805cf6c`)

**Decision (2026-11-13):** the SPIFFE ID layout of agents is configurable with two templates, one
for Klusterlet agents and one for add-on agents. This answers the proposal's Open Question 1;
see `questions.md` Q7. The defaults are the proposal's scheme, so nothing changes for a hub
that doesn't set them.

### Task U1: add the template fields

**Input:** `operator/v1/types_clustermanager.go`, `SPIFFERegistrationConfig`.

**Change:** add two constants next to `SPIFFEAuthType`, and two fields after `agentTrustDomains`.

```go
const (
	// DefaultSPIFFEAgentIDTemplate is the SPIFFE ID layout of Klusterlet agents from
	// enhancement #199.
	DefaultSPIFFEAgentIDTemplate = "spiffe://{trustDomain}/ocm/cluster/{cluster}/agent/{role}"
	// DefaultSPIFFEAddOnAgentIDTemplate is the SPIFFE ID layout of add-on agents.
	DefaultSPIFFEAddOnAgentIDTemplate = "spiffe://{trustDomain}/ocm/cluster/{cluster}/addon/{addon}/agent/{agent}"
)
```

```go
	// agentIDTemplate is the layout of Klusterlet agent SPIFFE IDs. The hub reads the cluster
	// and the agent role from it. It must start with spiffe://{trustDomain}/, where
	// {trustDomain} is any of agentTrustDomains. Path placeholders: {cluster} and {role}
	// (required) and {namespace} (optional; the agent's namespace, never used for
	// authorization). Each placeholder fills a whole path segment and appears at most once,
	// and at least one segment is fixed text. The Klusterlets must use the same template.
	// +optional
	// +kubebuilder:default:="spiffe://{trustDomain}/ocm/cluster/{cluster}/agent/{role}"
	// +kubebuilder:validation:MaxLength=512
	// +kubebuilder:validation:Pattern=`^spiffe://\{trustDomain\}(/([A-Za-z0-9._-]+|\{[A-Za-z]+\}))+$`
	AgentIDTemplate string `json:"agentIDTemplate,omitempty"`

	// addOnAgentIDTemplate is the layout of add-on agent SPIFFE IDs. Path placeholders:
	// {cluster}, {addon} and {agent} (required) and {namespace} (optional; the add-on's
	// install namespace, never used for authorization). The same rules as agentIDTemplate
	// apply, and no SPIFFE ID may match both templates.
	// +optional
	// +kubebuilder:default:="spiffe://{trustDomain}/ocm/cluster/{cluster}/addon/{addon}/agent/{agent}"
	// +kubebuilder:validation:MaxLength=512
	// +kubebuilder:validation:Pattern=`^spiffe://\{trustDomain\}(/([A-Za-z0-9._-]+|\{[A-Za-z]+\}))+$`
	AddOnAgentIDTemplate string `json:"addOnAgentIDTemplate,omitempty"`
```

The pattern only checks the shape. Known and required placeholders, duplicates, at least one
fixed segment, and no overlap between the two templates can't be CEL rules here (cost budget,
Q4). The cluster-manager operator checks them with sdk-go's `idtemplate` package (Day 4) and sets
`Degraded` (Day 14).

**Helpers** in `funcs_clustermanager.go`, like `GetAgentTrustDomains`, for objects that were not
defaulted by the API server (unit tests, fake clients):

```go
// GetAgentIDTemplate returns agentIDTemplate, or the default when it is empty.
func (c *SPIFFERegistrationConfig) GetAgentIDTemplate() string {
	return cmp.Or(c.AgentIDTemplate, DefaultSPIFFEAgentIDTemplate)
}

// GetAddOnAgentIDTemplate returns addOnAgentIDTemplate, or the default when it is empty.
func (c *SPIFFERegistrationConfig) GetAddOnAgentIDTemplate() string {
	return cmp.Or(c.AddOnAgentIDTemplate, DefaultSPIFFEAddOnAgentIDTemplate)
}
```

### Task U2: tests

Add to `test/integration/api/clustermanager_spiffe_test.go`:

| Case | `spiffe` block | Expected |
| --- | --- | --- |
| Defaults | `trustDomain` only | both templates filled with the defaults |
| Custom with namespace | `agentIDTemplate: spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}` | accepted, kept as given |
| Wrong prefix | `agentIDTemplate: spiffe://ocm-hub.example.org/ocm/{cluster}/{role}` | refused, `should match` |
| Placeholder inside a segment | `agentIDTemplate: spiffe://{trustDomain}/ocm/cluster-{cluster}/{role}` | refused, `should match` |
| Bad character | `addOnAgentIDTemplate: spiffe://{trustDomain}/ocm/{cluster}/{addon}/{agent}?x` | refused, `should match` |

Unknown or missing placeholders (`{clutser}`, no `{role}`) pass the CRD and are the operator's
job; leave them to Day 14's tests.

Add a unit test for both helpers: empty returns the default, set returns the value.

**Run:** `dgo 'make update && make test-api-integration'`, then `make verify` with
`DGO_IMAGE=golang:1.26`, then commit:

```shell
git add -A && git commit -s -m ":sparkles: configurable SPIFFE ID templates for agents and add-on agents"
git push fork spire
```

Then move the `replace` lines in `sdk-go` and `ocm` to the new SHA.

## In simple words: what changed and why

Day 2 only changed the API, which is the list of settings an admin can write on the hub's
`ClusterManager`. No hub code reads these settings yet; that comes on later days. Nothing we added
holds a key, a certificate or a token: the new settings are only names, addresses and durations.

First we added a feature gate called `SPIFFERegistration`, which is an on/off switch that is off
by default. While it is off, OCM behaves exactly as before. We did this because the feature is
new, and a hub that is already running must not change its behavior just because it was upgraded.
Then we added `authType: spiffe`, with a `spiffe` block that holds its settings. It tells the hub
that agents prove who they are with an SVID, a short-lived identity document issued by SPIRE. It
sits next to the existing drivers (`csr`, `awsirsa`, `grpc`) and doesn't replace them, so a hub
can keep its old clusters on the old drivers and add new ones with `spiffe`.

Inside the `spiffe` block, the first two settings are about trust domains. A trust domain is the
name of one SPIRE installation, for example `ocm-hub.example.org`. `trustDomain` is the hub's own:
the hub's identity comes from that SPIRE, so the hub has to know its name. `agentTrustDomains` is
the list of SPIREs the hub accepts agents from. A managed cluster can have its own SPIRE, separate
from the hub's, and the hub must only trust the ones the admin named, so an agent from any other
trust domain is refused. The hub's own trust domain is always accepted, even if the admin leaves
it out of the list; otherwise, forgetting to list it would lock out every agent that uses the
hub's SPIRE. The API server can't add it to the list for us, so we wrote one function,
`GetAgentTrustDomains()`, that returns the list with the hub's trust domain included. All later
code calls this function and never reads the raw list.

The next two settings are about the hub's own identity. `serverSPIFFEID` is the SPIFFE ID the hub
shows when an agent connects; if the admin doesn't set it, a standard one is used
(`spiffe://<trustDomain>/ocm/hub/grpc-server`). An agent checks this ID before it sends anything,
so it knows it is talking to the real hub, and that only works if the ID is fixed and known in
advance. `workloadAPISocket` is the path to the local SPIRE agent's socket inside the hub pod.
This socket is the only place the hub gets its SVID from, because OCM never creates or stores one.
The default is the path the SPIFFE CSI driver uses, so most admins won't set it.

Two time settings exist because of deleted clusters. `maxSVIDLifetime` (default `24h`) is the
longest an agent's SVID can be valid, and `maxClockSkew` (default `5m`) is how far the clocks of
the hub and the SPIRE servers may differ. When an admin deletes a cluster, an SVID issued a minute
earlier is still valid for hours, and with it the old agent could register again and bring the
cluster back. So the hub keeps a deleted cluster in "terminating" for the SVID lifetime plus the
clock skew. After that, no SVID from before the deletion can still be valid.

We also added `grpc-spiffe` as an endpoint protocol, which lets the hub expose a second gRPC port
only for the new driver. The new driver is a separate design from the existing `grpc` driver, and
with its own port nothing about the existing `grpc` port changes and both can run on the same hub.

Later the same day we made the layout of agent IDs configurable. `agentIDTemplate` and
`addOnAgentIDTemplate` describe the shape of an agent's SPIFFE ID, using placeholders such as
`{cluster}`, `{role}`, `{addon}` and `{agent}`. The hub reads the cluster name and the agent's
role out of the ID, so it must know where each part sits. Many companies already have their own
naming rules for SPIFFE IDs and can't use ours, and with templates they tell the hub their layout.
The defaults are the layout from the proposal, so a hub that doesn't set them behaves as before.

Finally, every field has rules, such as allowed characters, a maximum length, and "the `spiffe`
block is required when `authType` is `spiffe`". The API server refuses a setting that breaks a
rule, so a typo is caught at `kubectl apply` with a clear message, not hours later as a connection
that fails for no obvious reason. Two checks could not be done this way: that `serverSPIFFEID` is
inside `trustDomain`, and that the templates use the right placeholders. Kubernetes limits how
expensive a CRD's rules may be, and these go over the limit because the drivers are in a list with
no maximum size, so the operator will check them and report `Degraded` (Day 14). Each rule has a
test with one accepted value and one refused value, so if someone later loosens a rule by mistake,
a test fails.
