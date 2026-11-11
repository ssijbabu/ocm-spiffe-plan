# Day 3: API, Klusterlet, Klusterlet status and add-on driver

**Status:** done, except opening the upstream draft PR. api commits `a770a061` (Klusterlet, status,
add-on driver) and `c77b5174` (agent trust domain, pinning annotation), on `fork/spire`.

**Goal:** the `Klusterlet` accepts `authType: spiffe`, including agents in a trust domain other than
the hub's; add-ons can be switched to `spiffe`; the Klusterlet status can hold the trusted admission
state; `ManagedClusterAddOn` accepts the `spiffe` driver value; and the trust-domain pinning
annotation has a constant.

**Repo:** `api`, branch `spire` (Day 2's commits are already there).

## Before you start

| Input | Where |
| --- | --- |
| Klusterlet driver types | `operator/v1/types_klusterlet.go`: `RegistrationConfiguration`, `RegistrationDriver`, `AddOnRegistrationDriver`, `KlusterletStatus` |
| Add-on driver fields | `addon/v1beta1/types_managedclusteraddon.go` (`KubeClientConfig.Driver`) and `addon/v1alpha1/types_managedclusteraddon.go` (`Status.KubeClientDriver`) |
| ManagedCluster constants | `cluster/v1/types.go` |
| Design | proposal, API ("Managed cluster", "Add-ons"), "When the operator trusts the namespaces"; for trust domains, `spire-impl-by-mode`, "Managed cluster setup" and "Trust domain pinning" |

The code before Day 3:

```go
type RegistrationDriver struct {
	// +kubebuilder:validation:Enum=csr;awsirsa;grpc
	AuthType string   `json:"authType,omitempty"`
	AwsIrsa  *AwsIrsa `json:"awsIrsa,omitempty"`
}

type AddOnRegistrationDriver struct {
	// +kubebuilder:validation:Enum=csr;token
	AuthType string       `json:"authType,omitempty"`
	Token    *TokenConfig `json:"token,omitempty"`
}

// addon/v1beta1
type KubeClientConfig struct {
	Subject KubeClientSubject `json:"subject,omitempty"`
	// +kubebuilder:validation:Enum=csr;token
	Driver string `json:"driver,omitempty"`
}

// addon/v1alpha1, in ManagedClusterAddOnStatus
	// +kubebuilder:validation:Enum=csr;token
	KubeClientDriver string `json:"kubeClientDriver,omitempty"`
```

`RegistrationConfiguration.RegistrationDriver` is a value, not a pointer, and
`AddOnKubeClientRegistrationDriver` is a pointer. The CEL rules below use `has()` for both.

## Task 1: Klusterlet registration driver

**Change (as committed)** in `types_klusterlet.go`:

```go
// +kubebuilder:validation:XValidation:rule="!has(self.authType) || self.authType != 'spiffe' || has(self.spiffe)",message="spiffe is required when authType is spiffe"
type RegistrationDriver struct {
	// Type of the authentication used by managedcluster to register as well as pull work from hub.
	// Possible values are csr, awsirsa, grpc and spiffe.
	// +required
	// +kubebuilder:default:=csr
	// +kubebuilder:validation:Enum=csr;awsirsa;grpc;spiffe
	AuthType string `json:"authType,omitempty"`

	AwsIrsa *AwsIrsa `json:"awsIrsa,omitempty"`

	// spiffe contains what the agents need to reach the hub with their SPIRE-issued SVIDs.
	// It is required when authType is spiffe, and nothing in it is secret.
	// Effective only when the SPIFFERegistration feature gate is enabled.
	// +optional
	SPIFFE *KlusterletSPIFFEConfig `json:"spiffe,omitempty"`
}

// KlusterletSPIFFEConfig configures the spiffe registration driver on the managed cluster.
type KlusterletSPIFFEConfig struct {
	// hubEndpoint is host:port of the hub gRPC server's grpc-spiffe listener.
	// +required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=512
	HubEndpoint string `json:"hubEndpoint"`

	// hubSPIFFEID is the only SPIFFE ID the agents accept from the hub, for example
	// spiffe://ocm-hub.example.org/ocm/hub/grpc-server. It is in the hub's trust domain.
	// +required
	// +kubebuilder:validation:MaxLength=512
	// +kubebuilder:validation:Pattern=`^spiffe://[a-z0-9._-]+(/[A-Za-z0-9._-]+)+$`
	HubSPIFFEID string `json:"hubSPIFFEID"`

	// trustDomain is the agents' own SPIFFE trust domain, for example east.example.org. The
	// agents build their expected SPIFFE IDs in it, and the hub must list it in the
	// ClusterManager's agentTrustDomains. If it is not set, the trust domain of hubSPIFFEID is used.
	// +optional
	// +kubebuilder:validation:MaxLength=255
	// +kubebuilder:validation:Pattern=`^[a-z0-9._-]+$`
	TrustDomain string `json:"trustDomain,omitempty"`

	// workloadAPISocket is the address of the local SPIRE agent's Workload API, as mounted
	// into the agent pods by the SPIFFE CSI driver.
	// +optional
	// +kubebuilder:default:="unix:///spiffe-workload-api/spire-agent.sock"
	WorkloadAPISocket string `json:"workloadAPISocket,omitempty"`
}
```

And on `RegistrationConfiguration` (the struct holding both drivers):

```go
// +kubebuilder:validation:XValidation:rule="!has(self.registrationDriver) || !has(self.registrationDriver.authType) || self.registrationDriver.authType != 'spiffe' || (has(self.addOnKubeClientRegistrationDriver) && has(self.addOnKubeClientRegistrationDriver.authType) && self.addOnKubeClientRegistrationDriver.authType == 'spiffe')",message="addOnKubeClientRegistrationDriver.authType must be spiffe when registrationDriver.authType is spiffe"
type RegistrationConfiguration struct {
```

Field summary:

| Field | Default | Purpose |
| --- | --- | --- |
| `hubEndpoint` | none (required) | The hub gRPC server's SPIFFE listener, `host:port`. |
| `hubSPIFFEID` | none (required) | The only SPIFFE ID the agents accept from the hub. Always in the hub's trust domain. |
| `trustDomain` | the trust domain of `hubSPIFFEID` (filled by the operator) | The agents' own trust domain. The agents build their expected SPIFFE IDs in it. |
| `workloadAPISocket` | `unix:///spiffe-workload-api/spire-agent.sock` | The local SPIRE agent's Workload API. |

Unlike the ClusterManager (Day 2), these CEL rules are cheap enough: `registrationConfiguration`
isn't inside a list, so the CRD installs fine (`questions.md` Q5).

## Task 2: add-on driver

**Change (as committed):**

```go
type AddOnRegistrationDriver struct {
	// AuthType is the authentication driver used for add-on registration.
	// Possible values are csr, token and spiffe. With spiffe, each add-on agent authenticates
	// with its own SPIRE-issued SVID and no add-on credential is created.
	// ...
	// +optional
	// +kubebuilder:validation:Enum=csr;token;spiffe
	AuthType string `json:"authType,omitempty"`
	Token *TokenConfig `json:"token,omitempty"`
}
```

## Task 3: Klusterlet status

There will be three admission policies (agent namespaces, add-on namespaces, namespace labels;
Days 19 and 20), so the trusted state is a list keyed by policy name, not a single entry.

**Change (as committed):**

```go
type KlusterletStatus struct {
	// ... existing fields ...

	// spiffe records the state the klusterlet operator has made trusted when the
	// registration driver is spiffe. It holds only names, UIDs and generations.
	// +optional
	SPIFFE *KlusterletSPIFFEStatus `json:"spiffe,omitempty"`
}

const (
	// ConditionSPIFFEAdmissionReady is True once the klusterlet operator has finished its first
	// recovery of the agent and add-on namespaces and recorded the admission policies it trusts.
	// The platform should create the cluster's SPIRE registration entries only after that.
	ConditionSPIFFEAdmissionReady = "SPIFFEAdmissionReady"
)

type KlusterletSPIFFEStatus struct {
	// +optional
	// +listType=map
	// +listMapKey=policyName
	// +kubebuilder:validation:MaxItems=16
	AdmissionPolicies []SPIFFEAdmissionPolicyStatus `json:"admissionPolicies,omitempty"`

	// +optional
	AddOnNamespace *SPIFFENamespaceRef `json:"addOnNamespace,omitempty"`
}

type SPIFFEAdmissionPolicyStatus struct {
	PolicyName        string    `json:"policyName"`        // name of the policy and of its binding
	PolicyUID         types.UID `json:"policyUID"`
	PolicyGeneration  int64     `json:"policyGeneration"`
	BindingUID        types.UID `json:"bindingUID"`
	BindingGeneration int64     `json:"bindingGeneration"`
}

type SPIFFENamespaceRef struct {
	Name string    `json:"name"`
	UID  types.UID `json:"uid"`
}
```

`types` is `k8s.io/apimachinery/pkg/types`, added to the imports.

## Task 4: the trust-domain pinning annotation

**Change (as committed)** in `cluster/v1/types.go`, next to `ClusterImageRegistriesAnnotationKey`:

```go
	// SPIFFETrustDomainAnnotationKey pins a ManagedCluster that registers with the spiffe driver
	// to one SPIFFE trust domain. The hub accepts the cluster's Klusterlet and add-on agents only
	// with SVIDs from that trust domain. The hub sets it when the cluster is created and keeps it
	// on agent updates; an admin confirms or sets it when accepting the cluster.
	SPIFFETrustDomainAnnotationKey = "open-cluster-management.io/spiffe-trust-domain"
```

Only the constant is in the API. Setting and enforcing it is hub code (Day 11).

## Task 5: ManagedClusterAddOn driver value, both versions

Conversion maps v1beta1 `kubeClient.driver` to v1alpha1 `kubeClientDriver`, so both enums need
`spiffe`, or the value is refused in one version.

**Change (as committed):**

```go
// addon/v1beta1
	// driver is the authentication driver used by managedclusteraddon for kubeClient registration.
	// Possible values are csr, token and spiffe. With spiffe, the add-on agent authenticates with
	// its own SPIRE-issued SVID and no add-on credential is created.
	// +kubebuilder:validation:Enum=csr;token;spiffe
	Driver string `json:"driver,omitempty"`

// addon/v1alpha1
	// Supported values are `csr`, `token` and `spiffe`.
	// +kubebuilder:validation:Enum=csr;token;spiffe
	KubeClientDriver string `json:"kubeClientDriver,omitempty"`
```

## Task 6: regenerate and test

```shell
cd /home/ssijbabu/Documents/repos/ocm/api
dgo 'make update' && git status --short
```

**Output:**

```text
 M addon/v1alpha1/0000_01_addon.open-cluster-management.io_managedclusteraddons.crd.yaml
 M addon/v1alpha1/types_managedclusteraddon.go
 M addon/v1beta1/0000_01_addon.open-cluster-management.io_managedclusteraddons.crd.yaml
 M addon/v1beta1/types_managedclusteraddon.go
 M cluster/v1/types.go
 M operator/v1/0000_00_operator.open-cluster-management.io_klusterlets.crd.yaml
 M operator/v1/types_klusterlet.go
 M operator/v1/zz_generated.deepcopy.go
 M operator/v1/zz_generated.model_name.go
```

**Tests:** `test/integration/api/klusterlet_spiffe_test.go` (new).

| Spec | Input | Expected output |
| --- | --- | --- |
| spiffe + spiffe add-on driver | `hubEndpoint`, `hubSPIFFEID`, add-on `spiffe` | created; `workloadAPISocket` defaulted |
| agents in another trust domain | `trustDomain: east.example.org`, hub ID in `ocm-hub.example.org` | created, both stored |
| invalid agent trust domain | `trustDomain: East_Example` | refused |
| missing spiffe block | `authType: spiffe` only | refused: `spiffe is required when authType is spiffe` |
| csr add-on driver | spiffe + add-on `csr` | refused: `addOnKubeClientRegistrationDriver.authType must be spiffe ...` |
| no add-on driver | spiffe, no add-on driver | refused, same message |
| hub ID not a SPIFFE ID | `https://ocm-hub.example.org/...` | refused |
| empty hub endpoint | `hubEndpoint: ""` | refused |
| spiffe add-on driver alone | `authType: csr`, add-on `spiffe` | created |
| no registration configuration | `nil` | created (existing Klusterlets unaffected) |
| status | two `admissionPolicies`, `addOnNamespace`, `SPIFFEAdmissionReady=True` | stored and read back unchanged |
| duplicate status entry | same `policyName` twice | refused (`listType=map`) |
| add-on v1beta1 | `kubeClient.driver: spiffe` | accepted |
| add-on v1beta1, unknown | `kubeClient.driver: spire` | refused |
| add-on v1alpha1 | `kubeClientDriver: spiffe` | accepted and stored |

Example input for the "another trust domain" row:

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: cluster2
  deployOption:
    mode: Default
  registrationConfiguration:
    registrationDriver:
      authType: spiffe
      spiffe:
        hubEndpoint: ocm-grpc.hub.internal:443
        # Still the hub's ID, in the hub's trust domain
        hubSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        # The agents' own trust domain
        trustDomain: east.example.org
    addOnKubeClientRegistrationDriver:
      authType: spiffe
```

Stored object (relevant part):

```yaml
      spiffe:
        hubEndpoint: ocm-grpc.hub.internal:443
        hubSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        trustDomain: east.example.org
        workloadAPISocket: unix:///spiffe-workload-api/spire-agent.sock
```

**Found while implementing:** in envtest the add-on CRD stores v1alpha1 and there's no conversion
webhook, so v1beta1-only registration details (`kubeClient`) come back empty. The v1beta1 test only
checks that the value is accepted, as the existing token-driver test does; the v1alpha1 test checks
the stored value.

**Run:**

```shell
dgo 'make test-api-integration' 2>&1 | tail -2
```

**Output:**

```text
Ran 202 of 202 Specs in 5.9 seconds
SUCCESS! -- 202 Passed | 0 Failed | 0 Pending | 0 Skipped
```

## Task 7: verify, commit, push, draft PR

```shell
git add -A && git commit -s -m ":sparkles: add spiffe driver to Klusterlet and add-ons"
DGO_IMAGE=golang:1.26 dgo 'make verify' 2>&1 | tail -1     # 0 issues.
git push fork spire
```

**Output:** `fork/spire` has the Day 2 and Day 3 commits. The SHA that `sdk-go` and `ocm` point
their `replace` lines at is `git rev-parse HEAD`.

**Still to do:** open the upstream draft PR.

```shell
gh pr create -R open-cluster-management-io/api --draft --head ssijbabu:spire \
  --title ":sparkles: Add API for the spiffe registration driver" \
  --body "Implements the API part of enhancement #199 (open-cluster-management-io/enhancements#203)."
```

## Rules check

- [x] The Klusterlet spec has no bootstrap Secret reference for `spiffe`.
- [x] Status holds only names, UIDs and generations; nothing secret.
