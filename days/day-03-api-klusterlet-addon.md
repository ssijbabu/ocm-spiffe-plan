# Day 3: API, Klusterlet, Klusterlet status and add-on driver

**Goal:** the `Klusterlet` accepts `authType: spiffe`, add-ons can be switched to `spiffe`, the
Klusterlet status can hold the trusted admission state, and `ManagedClusterAddOn` accepts the
`spiffe` driver value. The `api` PR is opened as a draft.

**Repo:** `api`, branch `spire` (Day 2's commit is already there).

## Before you start

| Input | Where |
| --- | --- |
| Klusterlet driver types | `operator/v1/types_klusterlet.go`: `RegistrationDriver`, `AddOnRegistrationDriver`, `KlusterletStatus` |
| Add-on driver field | `addon/v1beta1/types_managedclusteraddon.go`: `KubeClientConfig.Driver` |
| Design | proposal, API ("Managed cluster" and "Add-ons"); "When the operator trusts the namespaces" |

Today's code:

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
```

## Task 1: Klusterlet registration driver

**Change** in `types_klusterlet.go`:

```go
type RegistrationDriver struct {
	// +kubebuilder:validation:Enum=csr;awsirsa;grpc;spiffe
	AuthType string   `json:"authType,omitempty"`
	AwsIrsa  *AwsIrsa `json:"awsIrsa,omitempty"`

	// spiffe holds what the agents need to reach the hub with their SVIDs.
	// Required when authType is spiffe. Nothing here is secret.
	// +optional
	SPIFFE *KlusterletSPIFFEConfig `json:"spiffe,omitempty"`
}

type KlusterletSPIFFEConfig struct {
	// hubEndpoint is host:port of the hub's grpc-spiffe listener.
	// +required
	// +kubebuilder:validation:MinLength=1
	HubEndpoint string `json:"hubEndpoint"`

	// hubSPIFFEID is the only SPIFFE ID the agents accept from the hub.
	// +required
	// +kubebuilder:validation:Pattern=`^spiffe://[a-z0-9._-]+(/[A-Za-z0-9._-]+)+$`
	HubSPIFFEID string `json:"hubSPIFFEID"`

	// workloadAPISocket is the address of the local SPIRE agent's Workload API.
	// +optional
	// +kubebuilder:default:="unix:///spiffe-workload-api/spire-agent.sock"
	WorkloadAPISocket string `json:"workloadAPISocket,omitempty"`
}
```

Add CEL rules on `RegistrationConfiguration` (the struct that holds both drivers):

```go
// +kubebuilder:validation:XValidation:rule="!has(self.registrationDriver) || self.registrationDriver.authType != 'spiffe' || has(self.registrationDriver.spiffe)",message="spiffe is required when authType is spiffe"
// +kubebuilder:validation:XValidation:rule="!has(self.registrationDriver) || self.registrationDriver.authType != 'spiffe' || (has(self.addOnKubeClientRegistrationDriver) && self.addOnKubeClientRegistrationDriver.authType == 'spiffe')",message="addOnKubeClientRegistrationDriver.authType must be spiffe when registrationDriver.authType is spiffe"
```

## Task 2: add-on driver

```go
type AddOnRegistrationDriver struct {
	// Possible values are csr, token and spiffe.
	// +kubebuilder:validation:Enum=csr;token;spiffe
	AuthType string `json:"authType,omitempty"`
	Token *TokenConfig `json:"token,omitempty"`
}
```

## Task 3: Klusterlet status

**Change:**

```go
type KlusterletStatus struct {
	// ... existing fields ...

	// spiffe records the state the klusterlet operator has made trusted in spiffe mode.
	// +optional
	SPIFFE *KlusterletSPIFFEStatus `json:"spiffe,omitempty"`
}

type KlusterletSPIFFEStatus struct {
	// admissionPolicy is the admission policy and binding the operator ran recovery for.
	// +optional
	AdmissionPolicy *SPIFFEAdmissionPolicyStatus `json:"admissionPolicy,omitempty"`

	// addOnNamespace is the add-on namespace the operator created, by name and UID.
	// +optional
	AddOnNamespace *SPIFFENamespaceRef `json:"addOnNamespace,omitempty"`
}

type SPIFFEAdmissionPolicyStatus struct {
	PolicyUID         types.UID `json:"policyUID"`
	PolicyGeneration  int64     `json:"policyGeneration"`
	BindingUID        types.UID `json:"bindingUID"`
	BindingGeneration int64     `json:"bindingGeneration"`
}

type SPIFFENamespaceRef struct {
	Name string    `json:"name"`
	UID  types.UID `json:"uid"`
}

const (
	// ConditionSPIFFEAdmissionReady is True once the first recovery has finished and the
	// admission state is recorded. The platform creates SPIRE entries only after that.
	ConditionSPIFFEAdmissionReady = "SPIFFEAdmissionReady"
)
```

The operator can be split into several policies (agent namespaces, add-on namespaces, namespace
labels). If so, make `AdmissionPolicy` a list keyed by policy name. Decide when you write Day 19
and adjust here; it is still a draft PR.

## Task 4: ManagedClusterAddOn driver value

In `addon/v1beta1/types_managedclusteraddon.go`:

```go
	// driver is the authentication driver used by managedclusteraddon for kubeClient registration.
	// Possible values are csr, token and spiffe.
	// +kubebuilder:validation:Enum=csr;token;spiffe
	Driver string `json:"driver,omitempty"`
```

Check whether `addon/v1alpha1` has the same field with conversion. If it does, the conversion
must keep `spiffe` (round-trip test).

## Task 5: regenerate and test

```shell
cd /home/ssijbabu/Documents/repos/ocm/api
dgo 'make update' && git status --short
```

**Output (files changed):**

```text
 M addon/v1beta1/0000_01_addon.open-cluster-management.io_managedclusteraddons.crd.yaml
 M addon/v1beta1/types_managedclusteraddon.go
 M operator/v1/0000_00_operator.open-cluster-management.io_klusterlets.crd.yaml
 M operator/v1/types_klusterlet.go
 M operator/v1/zz_generated.deepcopy.go
 ...
```

Add envtest cases to `test/integration/api/klusterlet_test.go`:

| Input Klusterlet | Expected result |
| --- | --- |
| `authType: spiffe`, `spiffe` set, add-on driver `spiffe` | Created; `workloadAPISocket` defaulted |
| `authType: spiffe`, no `spiffe` block | Refused: `spiffe is required when authType is spiffe` |
| `authType: spiffe`, add-on driver `csr` | Refused: `addOnKubeClientRegistrationDriver.authType must be spiffe ...` |
| `hubSPIFFEID: https://hub` | Refused by the pattern |
| `authType: csr`, add-on driver `spiffe` | Created (allowed; the agents decide at runtime) |
| Status update with `spiffe.admissionPolicy` | Stored and read back unchanged |

Example test input (the first row):

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: cluster1
  deployOption:
    mode: Default
  registrationConfiguration:
    registrationDriver:
      authType: spiffe
      spiffe:
        hubEndpoint: ocm-grpc.hub.internal:443
        hubSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
    addOnKubeClientRegistrationDriver:
      authType: spiffe
```

Expected stored object (relevant part):

```yaml
    registrationDriver:
      authType: spiffe
      spiffe:
        hubEndpoint: ocm-grpc.hub.internal:443
        hubSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
        workloadAPISocket: unix:///spiffe-workload-api/spire-agent.sock
```

**Run:**

```shell
dgo 'make test-api-integration' 2>&1 | tail -2
dgo 'make verify' 2>&1 | tail -3
```

**Output:** all specs pass, and `make verify` exits 0.

## Task 6: commit, push and open a draft PR

```shell
git add -A && git commit -s -m ":sparkles: add spiffe driver to Klusterlet and add-ons

Add authType spiffe with hubEndpoint, hubSPIFFEID and workloadAPISocket,
require the spiffe add-on driver with it, add the Klusterlet spiffe status
(admission policy UIDs and generations, add-on namespace) and the
SPIFFEAdmissionReady condition, and allow driver spiffe on ManagedClusterAddOn."
git push fork spire
gh pr create -R open-cluster-management-io/api --draft --head ssijbabu:spire \
  --title ":sparkles: Add API for the spiffe registration driver" \
  --body "Implements the API part of enhancement #199 (PR open-cluster-management-io/enhancements#203)."
```

**Output:** two commits on `fork/spire`, and a draft PR URL. Note the head commit SHA; `sdk-go` and
`ocm` point at it with `replace` lines:

```shell
git rev-parse HEAD   # e.g. 4f2a9c1e...
```

## Rules check

- [ ] The Klusterlet spec has no bootstrap Secret reference for `spiffe`.
- [ ] Status holds only UIDs, generations and names; nothing secret.
