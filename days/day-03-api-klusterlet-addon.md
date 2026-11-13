# Day 3: API, Klusterlet, Klusterlet status and add-on driver

**Status:** done, except opening the upstream draft PR. "Update: agent ID templates" is api `b805cf6c` (2026-11-13), pushed to `fork/spire`. api commits `a770a061` (Klusterlet, status,
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


**Every validation rule has a test.** For each rule on the new Klusterlet, status and add-on fields:

| Field | Rule | Accepted case | Refused case |
| --- | --- | --- | --- |
| `RegistrationDriver` | CEL `has(self.spiffe)` | spiffe + settings | missing spiffe block |
| `RegistrationConfiguration` | CEL add-on driver must be `spiffe` | spiffe + add-on `spiffe`; csr + add-on `spiffe` | add-on `csr`; no add-on driver; add-on driver without `authType` |
| `hubEndpoint` | required, minLength 1 | set | empty |
| `hubEndpoint` | maxLength 512 | | over 512 |
| `hubSPIFFEID` | pattern | hub ID | `https://...` |
| `hubSPIFFEID` | maxLength 512 | | over 512 |
| `trustDomain` | optional | not set (stays empty) | |
| `trustDomain` | pattern | `east.example.org` | `East_Example` |
| `trustDomain` | maxLength 255 | | 256 characters |
| `workloadAPISocket` | default | defaulted | |
| add-on `authType` | enum adds `spiffe` | `spiffe` | |
| `status.spiffe.admissionPolicies` | `listType=map` on `policyName` | two names | same name twice |
| `status.spiffe.admissionPolicies` | maxItems 16 | 16 | 17 |
| `policyName` | minLength 1, maxLength 253 | `ocm-spiffe-agents-klusterlet` | empty; 254 characters |
| `addOnNamespace.name` | minLength 1, maxLength 63 | `open-cluster-management-agent-addon` | empty; 64 characters |
| `ManagedClusterAddOn` v1beta1 `kubeClient.driver` | enum adds `spiffe` | `spiffe` | `spire` |
| `ManagedClusterAddOn` v1alpha1 `kubeClientDriver` | enum adds `spiffe` | `spiffe` | (existing `invalid-driver` test) |

`SPIFFETrustDomainAnnotationKey` and `ConditionSPIFFEAdmissionReady` are constants; the hub and
operator code that uses them is tested on Days 11 and 21.

**Run:**

```shell
dgo 'make test-api-integration' 2>&1 | tail -2
```

**Output:**

```text
Ran 223 of 223 Specs in 5.6 seconds
SUCCESS! -- 223 Passed | 0 Failed | 0 Pending | 0 Skipped
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

## Update: agent ID templates (done: api `b805cf6c`)

The agents must build the same IDs the hub parses, so the Klusterlet gets the same two templates
as the ClusterManager (Day 2, "Update: agent ID templates"). The Klusterlet operator renders each
agent's full expected ID from them (Day 18); agents never parse templates. The add-on template is
here because add-on agents run on the managed cluster too, and get their expected ID from the
same place as `hubEndpoint` and `hubSPIFFEID` (Day 23, `questions.md` Q8).

### Task U1: add the fields

**Input:** `operator/v1/types_klusterlet.go`, `KlusterletSPIFFEConfig`, after `trustDomain`.

```go
	// agentIDTemplate is the layout of this Klusterlet's agent SPIFFE IDs. It must equal the
	// hub's ClusterManager agentIDTemplate, or the hub refuses the agents. {trustDomain} is
	// trustDomain, {cluster} the cluster name, {role} registration, work or klusterlet, and
	// {namespace} the namespace the agent runs in.
	// +optional
	// +kubebuilder:default:="spiffe://{trustDomain}/ocm/cluster/{cluster}/agent/{role}"
	// +kubebuilder:validation:MaxLength=512
	// +kubebuilder:validation:Pattern=`^spiffe://\{trustDomain\}(/([A-Za-z0-9._-]+|\{[A-Za-z]+\}))+$`
	AgentIDTemplate string `json:"agentIDTemplate,omitempty"`

	// addOnAgentIDTemplate is the layout of add-on agent SPIFFE IDs on this cluster. It must
	// equal the hub's ClusterManager addOnAgentIDTemplate. {namespace} is the add-on's install
	// namespace.
	// +optional
	// +kubebuilder:default:="spiffe://{trustDomain}/ocm/cluster/{cluster}/addon/{addon}/agent/{agent}"
	// +kubebuilder:validation:MaxLength=512
	// +kubebuilder:validation:Pattern=`^spiffe://\{trustDomain\}(/([A-Za-z0-9._-]+|\{[A-Za-z]+\}))+$`
	AddOnAgentIDTemplate string `json:"addOnAgentIDTemplate,omitempty"`
```

CEL works on the Klusterlet (Q5), so check the required placeholders in the CRD as well, on
`KlusterletSPIFFEConfig`:

```go
// +kubebuilder:validation:XValidation:rule="!has(self.agentIDTemplate) || (self.agentIDTemplate.contains('/{cluster}') && self.agentIDTemplate.contains('/{role}'))",message="agentIDTemplate must contain {cluster} and {role}"
// +kubebuilder:validation:XValidation:rule="!has(self.addOnAgentIDTemplate) || (self.addOnAgentIDTemplate.contains('/{cluster}') && self.addOnAgentIDTemplate.contains('/{addon}') && self.addOnAgentIDTemplate.contains('/{agent}'))",message="addOnAgentIDTemplate must contain {cluster}, {addon} and {agent}"
```

Unknown placeholders, duplicates and overlap are checked by the Klusterlet operator with sdk-go's
`idtemplate` package (Day 18), which sets `Degraded`.

Add `GetAgentIDTemplate()` and `GetAddOnAgentIDTemplate()` on `KlusterletSPIFFEConfig`, as on the
ClusterManager, reusing the same default constants.

**`clusterName` becomes required with `spiffe`.** The operator renders each agent's ID before the
agent runs, so it can't wait for the registration agent to generate a cluster name. The SPIRE
entries need the literal name anyway. On `KlusterletSpec`:

```go
// +kubebuilder:validation:XValidation:rule="!has(self.registrationConfiguration) || !has(self.registrationConfiguration.registrationDriver) || !has(self.registrationConfiguration.registrationDriver.authType) || self.registrationConfiguration.registrationDriver.authType != 'spiffe' || (has(self.clusterName) && self.clusterName != '')",message="clusterName is required when registrationDriver.authType is spiffe"
```

Check the real field path from `KlusterletSpec` to `registrationDriver` before writing the rule.

### Task U2: tests

Add to the Klusterlet SPIFFE integration tests:

| Case | `spiffe` block | Expected |
| --- | --- | --- |
| Defaults | `hubEndpoint`, `hubSPIFFEID` only | both templates filled with the defaults |
| Custom with namespace | `agentIDTemplate: spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}` | accepted |
| No `{role}` | `agentIDTemplate: spiffe://{trustDomain}/ocm/cluster/{cluster}/agent` | refused, `must contain {cluster} and {role}` |
| No `{agent}` | `addOnAgentIDTemplate: spiffe://{trustDomain}/ocm/{cluster}/{addon}` | refused, `must contain {cluster}, {addon} and {agent}` |
| Wrong prefix | `agentIDTemplate: spiffe://east.example.org/ocm/{cluster}/{role}` | refused, `should match` |
| No `clusterName` | `authType: spiffe`, `clusterName` empty | refused, `clusterName is required` |
| `clusterName` set | `authType: spiffe`, `clusterName: cluster1` | accepted |

**Run, verify and commit** as in Task 7, in the same commit as Day 2's update.

### Result (2026-11-13)

Committed as api `b805cf6c` (with Day 2's update) and pushed to `fork/spire`:

- ClusterManager and Klusterlet: `agentIDTemplate`, `addOnAgentIDTemplate`, the default constants
  and `Get...` helpers; Klusterlet CEL rules for the required placeholders.
- Klusterlet: `clusterName` required with `spiffe`. The rule uses `size(self.clusterName) > 0`,
  not `self.clusterName != ''`: gofmt turns `''` in a doc comment into a typographic quote, which
  breaks the generated rule.
- Klusterlet status: `status.spiffe.agentIDs`, a list keyed by `role` (`registration`, `work`,
  `klusterlet`), at most 3.
- Tests: the table cases are loops of `It` (Ginkgo v1's `extensions/table` isn't vendored).
  `make test-api-integration`: 247 of 247 (was 223). Unit tests for the four helpers pass.
- `make verify` on the committed tree (Go 1.26): 0 issues. (Before committing,
  `verify-codegen-crds` fails only because it ends with `git diff --exit-code`.)
- `sdk-go` points at it (`replace ... => github.com/ssijbabu/api v1.3.1-0.20261113090800-b805cf6c07e4`).
  `ocm` moves to it on Day 10.

## In simple words: what changed and why

Day 3 did for the managed cluster what Day 2 did for the hub. It only changed the API, which is
the list of settings an admin can write on the `Klusterlet`. No agent code reads them yet.

### The agents' side of the new driver

We added `authType: spiffe` to the Klusterlet, with a `spiffe` block for its settings. It tells
the agents to reach the hub with their SPIRE-issued SVID.

In the other drivers, the agents start from a bootstrap kubeconfig Secret. In this mode there is
none: the SVID is the only credential.

### Where the hub is, and how to recognize it

`hubEndpoint` is the address of the hub's `grpc-spiffe` port, so the agents know where to connect.

`hubSPIFFEID` is the only identity the agents accept from the hub. When an agent connects, the
server shows its SPIFFE ID. If it isn't exactly this one, the agent drops the connection. This
protects the agent from a fake hub.

### The agents' own SPIRE

`trustDomain` is the name of the SPIRE the agents get their SVIDs from. A managed cluster can have
its own SPIRE, different from the hub's, and the agents build their own SPIFFE ID with this name.
If it is not set, the hub's trust domain is used, which fits the simple setup where everyone
shares one SPIRE.

`workloadAPISocket` is the path to the local SPIRE agent's socket inside the agent pods. As on the
hub, it is the only place an agent gets its SVID from.

### Add-ons must use SPIFFE too

We added a rule: when the Klusterlet uses `spiffe`, the add-on driver must also be `spiffe`.

With the `csr` or `token` add-on drivers, OCM creates a credential for each add-on and stores it
in a Secret. The requirement is that OCM stores no credentials at all, so mixing isn't allowed.

For the same reason `spiffe` is now an accepted driver value on `ManagedClusterAddOn`. It had to
be added to both API versions (v1alpha1 and v1beta1), because Kubernetes converts one into the
other and would refuse the value in the version that didn't know it.

### The operator writes down what it has protected

SPIRE gives an SVID to any pod that runs in certain namespaces with a certain ServiceAccount. So
before SPIRE is told about a cluster, the operator must make sure only the right pods can run
there. It does that with admission policies.

`status.spiffe` is where the operator records this: the policies it installed and the add-on
namespace it created, each by name and UID. The UID matters because a name can be reused. If
someone deletes a policy and creates another with the same name, the UID is different and the
operator notices.

The `SPIFFEAdmissionReady` condition is the signal to the platform: once it is True, it is safe to
create the cluster's SPIRE entries.

### Pinning a cluster to one SPIRE

We added a constant for the annotation `open-cluster-management.io/spiffe-trust-domain` on the
`ManagedCluster`. It records which trust domain that cluster's agents must come from.

The hub can trust several SPIREs. Without the pin, an admin of the "west" SPIRE could issue an
SVID with the name of a cluster that belongs to "east", and take that cluster over. With the pin,
the hub accepts `cluster2` only from the trust domain it was registered with.

Day 3 only added the name of the annotation. The hub code that sets and checks it comes on
Day 11.

### The same ID layouts as the hub

The Klusterlet got the same two templates as the ClusterManager, `agentIDTemplate` and
`addOnAgentIDTemplate`. The agents must show exactly the ID the hub expects, so both sides need
the same layout. If they differ, the hub refuses the agents.

The API server refuses a template that leaves out a needed part: `{cluster}` and `{role}` for
agents; `{cluster}`, `{addon}` and `{agent}` for add-ons. Without those parts the hub couldn't
tell which cluster or which agent is calling. On the hub this check had to be left to the operator
(see Day 2), but on the Klusterlet the rule is cheap enough for the API server.

### The cluster name is now required

With `spiffe`, `clusterName` must be set. In the other drivers it can be left empty and the
registration agent generates one.

The cluster name is part of the agent's SPIFFE ID. The operator has to build that ID before the
agent starts, and the SPIRE entries need the real name too, so a name generated later is too late.

### The operator shows the exact agent IDs

`status.spiffe.agentIDs` lists the full SPIFFE ID of each agent (registration, work, klusterlet),
as the operator built it from the template.

Whoever creates the SPIRE entries has to give SPIRE exactly these IDs. They can copy them from the
status and don't have to fill in the template by hand, where a small mistake would make the hub
refuse the agent.

### A test for every rule

As on Day 2, each rule has a test with one accepted value and one refused value.

### What did not change

The Klusterlet has no Secret reference for `spiffe`, and the status holds only names, UIDs,
generations and IDs. A Klusterlet that doesn't use `spiffe` is not affected.
