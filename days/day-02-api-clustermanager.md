# Day 2: API, ClusterManager driver and feature gate

**Goal:** the `ClusterManager` accepts `authType: spiffe` with its settings, the hub can expose a
`grpc-spiffe` endpoint, and the `SPIFFERegistration` feature gate exists.

**Repo:** `api`, branch `spire`.

## Before you start

| Input | Where |
| --- | --- |
| Driver types | `operator/v1/types_clustermanager.go`: `RegistrationDriverHub`, `EndpointExposure`, auth type constants |
| Feature gates | `feature/feature.go` |
| Design | proposal, section API ("Hub") |

Today's code (what you are extending):

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
	// SPIFFERegistration enables the spiffe registration driver: agents and add-on agents
	// authenticate to the hub gRPC server with SPIRE-issued X.509-SVIDs only.
	SPIFFERegistration featuregate.Feature = "SPIFFERegistration"
```

Add it, `{Default: false, PreRelease: featuregate.Alpha}`, to:

- `DefaultHubRegistrationFeatureGates`, used by the hub controllers and gRPC server;
- `DefaultSpokeRegistrationFeatureGates`, used by the registration agent;
- `DefaultSpokeWorkFeatureGates`, used by the work agent;
- `DefaultHubAddonManagerFeatureGates`, used by the add-on manager.

**Output:** `go build ./...` passes. The gate is off by default everywhere.

## Task 2: add the auth type and the spiffe block

**Input:** `operator/v1/types_clustermanager.go`.

**Change:**

```go
const (
	// SPIFFEAuthType represents the authentication type that uses SPIRE-issued SVIDs.
	SPIFFEAuthType = "spiffe"
)

type RegistrationDriverHub struct {
	// authType is the type of the authentication used by hub to initialize the Hub cluster.
	// Possible values are csr, awsirsa, grpc and spiffe.
	// +required
	// +kubebuilder:default:=csr
	// +kubebuilder:validation:Enum=csr;awsirsa;grpc;spiffe
	AuthType string `json:"authType,omitempty"`
	// ... existing fields ...

	// spiffe represents the configuration for the spiffe driver.
	// +optional
	SPIFFE *SPIFFERegistrationConfig `json:"spiffe,omitempty"`
}

// SPIFFERegistrationConfig configures the spiffe registration driver on the hub.
// +kubebuilder:validation:XValidation:rule="!has(self.serverSPIFFEID) || self.serverSPIFFEID.startsWith('spiffe://' + self.trustDomain + '/')",message="serverSPIFFEID must be in trustDomain"
type SPIFFERegistrationConfig struct {
	// trustDomain is the SPIFFE trust domain of the hub and its agents.
	// +required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=255
	// +kubebuilder:validation:Pattern=`^[a-z0-9._-]+$`
	TrustDomain string `json:"trustDomain"`

	// serverSPIFFEID is the SPIFFE ID the hub gRPC server presents.
	// Default: spiffe://<trustDomain>/ocm/hub/grpc-server
	// +optional
	// +kubebuilder:validation:Pattern=`^spiffe://[a-z0-9._-]+(/[A-Za-z0-9._-]+)+$`
	ServerSPIFFEID string `json:"serverSPIFFEID,omitempty"`

	// workloadAPISocket is the address of the local SPIRE agent's Workload API.
	// +optional
	// +kubebuilder:default:="unix:///spiffe-workload-api/spire-agent.sock"
	WorkloadAPISocket string `json:"workloadAPISocket,omitempty"`

	// maxSVIDLifetime is the longest SVID TTL SPIRE issues to OCM agents. The delete guard
	// waits this long after a ManagedCluster is deleted.
	// +optional
	// +kubebuilder:default:="24h"
	MaxSVIDLifetime metav1.Duration `json:"maxSVIDLifetime,omitempty"`

	// maxClockSkew is the allowed clock difference between SPIRE and the hub.
	// +optional
	// +kubebuilder:default:="5m"
	MaxClockSkew metav1.Duration `json:"maxClockSkew,omitempty"`
}
```

The trust-domain pattern follows the SPIFFE spec: lowercase letters, digits, `.`, `-` and `_`.

## Task 3: allow `grpc-spiffe` endpoint exposure

**Input:** `EndpointExposure` in the same file.

**Change:**

```go
	// protocol is the protocol used for the endpoint, could be https, grpc or grpc-spiffe.
	// grpc-spiffe is the separate listener of the spiffe registration driver.
	// +kubebuilder:default:=grpc
	// +kubebuilder:validation:Enum=grpc;https;grpc-spiffe
	// +required
	Protocol string `json:"protocol"`
```

The `grpc-spiffe` entry reuses the `grpc` field for its exposure type (hostname, loadBalancer or
route), so no new field is needed. Write that in the field comment.

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
 M operator/v1/zz_generated.swagger_doc_generated.go
```

Check the CRD has the new enum value and CEL rule:

```shell
grep -n 'grpc-spiffe\|spiffe:\|serverSPIFFEID must be' operator/v1/0000_01_operator.open-cluster-management.io_clustermanagers.crd.yaml
```

## Task 5: integration tests for the ClusterManager

**Input:** `test/integration/api/clustermanager_test.go` (envtest, Ginkgo).

**Add:**

```go
ginkgo.Context("spiffe registration driver", func() {
	ginkgo.It("accepts a minimal spiffe driver and fills the defaults", func() {
		cm := newClusterManager("cm-spiffe-1")
		cm.Spec.RegistrationConfiguration = &operatorv1.RegistrationHubConfiguration{
			RegistrationDrivers: []operatorv1.RegistrationDriverHub{{
				AuthType: operatorv1.SPIFFEAuthType,
				SPIFFE:   &operatorv1.SPIFFERegistrationConfig{TrustDomain: "ocm-hub.example.org"},
			}},
		}
		created, err := operatorClient.OperatorV1().ClusterManagers().Create(ctx, cm, metav1.CreateOptions{})
		gomega.Expect(err).ToNot(gomega.HaveOccurred())
		s := created.Spec.RegistrationConfiguration.RegistrationDrivers[0].SPIFFE
		gomega.Expect(s.WorkloadAPISocket).To(gomega.Equal("unix:///spiffe-workload-api/spire-agent.sock"))
		gomega.Expect(s.MaxSVIDLifetime.Duration).To(gomega.Equal(24 * time.Hour))
		gomega.Expect(s.MaxClockSkew.Duration).To(gomega.Equal(5 * time.Minute))
	})

	ginkgo.It("refuses a server ID outside the trust domain", func() {
		cm := newClusterManager("cm-spiffe-2")
		cm.Spec.RegistrationConfiguration = &operatorv1.RegistrationHubConfiguration{
			RegistrationDrivers: []operatorv1.RegistrationDriverHub{{
				AuthType: operatorv1.SPIFFEAuthType,
				SPIFFE: &operatorv1.SPIFFERegistrationConfig{
					TrustDomain:    "ocm-hub.example.org",
					ServerSPIFFEID: "spiffe://other.example.org/ocm/hub/grpc-server",
				},
			}},
		}
		_, err := operatorClient.OperatorV1().ClusterManagers().Create(ctx, cm, metav1.CreateOptions{})
		gomega.Expect(err).To(gomega.MatchError(gomega.ContainSubstring("serverSPIFFEID must be in trustDomain")))
	})

	ginkgo.It("refuses an invalid trust domain", func() { /* "OCM_Hub" -> pattern error */ })
	ginkgo.It("accepts a grpc-spiffe endpoint exposure", func() { /* Protocol: "grpc-spiffe" */ })
})
```

Check the real helper names (`newClusterManager`, the API type name for
`RegistrationConfiguration`) in the test file first, and use those.

**Run:**

```shell
dgo 'make test-api-integration' 2>&1 | tail -3
```

**Output:**

```text
Ran 184 of 184 Specs in 6.5 seconds
SUCCESS! -- 184 Passed | 0 Failed | 0 Pending | 0 Skipped
```

That's the 180 specs from the baseline plus the 4 new ones.

## Task 6: verify and commit

```shell
dgo 'make verify' 2>&1 | tail -5
git add -A && git commit -s -m ":sparkles: add spiffe registration driver to ClusterManager

Add authType spiffe with trustDomain, serverSPIFFEID, workloadAPISocket,
maxSVIDLifetime and maxClockSkew, the grpc-spiffe endpoint protocol, and the
SPIFFERegistration feature gate (alpha, off by default)."
```

**Output:** one commit. Don't push yet; Day 3 adds the Klusterlet part to the same PR.

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
        trustDomain: ocm-hub.example.org
  serverConfiguration:
    endpointsExposure:
    - protocol: grpc-spiffe
      grpc:
        type: loadBalancer
```

After it is applied, `kubectl get clustermanager cluster-manager -o yaml` shows the defaults filled
in: `serverSPIFFEID` stays empty (the operator computes it), `workloadAPISocket`,
`maxSVIDLifetime: 24h0m0s` and `maxClockSkew: 5m0s`.

## Rules check

- [ ] No field holds a key, certificate or token.
- [ ] Everything is behind `SPIFFERegistration`.
