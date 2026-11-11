# Day 18: the Klusterlet operator in all four install modes

**Goal:** with `authType: spiffe` on the Klusterlet, the operator deploys the agents with the
Workload API volume, the right `spiffe-role` label and ServiceAccount, and the SPIFFE flags, in
`Default`, `Singleton`, `Hosted` and `SingletonHosted`. It creates no bootstrap or hub-kubeconfig
Secret.

**Repo:** `ocm`, branch `spire`, `pkg/operator/operators/klusterlet` and `manifests/klusterlet`.

## Before you start

| Input | Where |
| --- | --- |
| Agent Deployment templates | `manifests/klusterlet/management/klusterlet-registration-deployment.yaml`, `klusterlet-work-deployment.yaml`, `klusterlet-agent-deployment.yaml` |
| Config struct | `KlusterletConfig` in `pkg/operator/operators/klusterlet/controllers/klusterletcontroller/klusterlet_controller.go` |
| Pairs table | proposal, Admission policy (role label and ServiceAccount per mode) |

Today the work Deployment template already branches on the driver:

```yaml
          {{if eq .RegistrationDriver.AuthType "grpc"}}
          ...
          - "--cloudevents-client-id={{ .ClusterName }}-work-agent"
          {{else}}
          ...
          {{end}}
```

The expected result per mode, for a Klusterlet named `klusterlet` (hosted: named `cluster4`):

| Mode | Deployment | Namespace | ServiceAccount | `spiffe-role` |
| --- | --- | --- | --- | --- |
| `Default` | `klusterlet-registration-agent` | `open-cluster-management-agent` | `klusterlet-registration-sa` | `registration` |
| `Default` | `klusterlet-work-agent` | `open-cluster-management-agent` | `klusterlet-work-sa` | `work` |
| `Singleton` | `klusterlet-agent` | `open-cluster-management-agent` | `klusterlet-work-sa` | `klusterlet` |
| `Hosted` | `cluster4-registration-agent` | `cluster4` (on the management cluster) | `cluster4-registration-sa` | `registration` |
| `Hosted` | `cluster4-work-agent` | `cluster4` | `cluster4-work-sa` | `work` |
| `SingletonHosted` | `cluster4-agent` | `cluster4` | `cluster4-work-sa` | `klusterlet` |

Check the singleton's ServiceAccount in `klusterlet-agent-deployment.yaml`; the proposal says it
reuses `<klusterlet>-work-sa`.

## Task 1: config

```go
type KlusterletConfig struct {
	// ... existing ...
	SPIFFEHubEndpoint       string
	SPIFFEHubID             string
	SPIFFEWorkloadAPISocket string
}
```

Fill from `klusterlet.Spec.RegistrationConfiguration.RegistrationDriver.SPIFFE`.

## Task 2: templates

In each of the three Deployment templates, add (example for the work agent):

```yaml
  template:
    metadata:
      labels:
        app: klusterlet-manifestwork-agent
        {{ if eq .RegistrationDriver.AuthType "spiffe" }}
        open-cluster-management.io/spiffe-role: work
        {{ end }}
    spec:
      containers:
      - name: klusterlet-manifestwork-agent
        args:
          {{ if eq .RegistrationDriver.AuthType "spiffe" }}
          - "--spiffe-hub-endpoint={{ .SPIFFEHubEndpoint }}"
          - "--spiffe-hub-id={{ .SPIFFEHubID }}"
          - "--spiffe-workload-api-socket={{ .SPIFFEWorkloadAPISocket }}"
          {{ end }}
        volumeMounts:
          {{ if eq .RegistrationDriver.AuthType "spiffe" }}
          - name: spiffe-workload-api
            mountPath: /spiffe-workload-api
            readOnly: true
          {{ else }}
          - name: hub-kubeconfig-secret   # existing mount, only for other drivers
            ...
          {{ end }}
      volumes:
        {{ if eq .RegistrationDriver.AuthType "spiffe" }}
        - name: spiffe-workload-api
          csi:
            driver: csi.spiffe.io
            readOnly: true
        {{ end }}
```

The registration Deployment uses `spiffe-role: registration`; the singleton uses `klusterlet`.

## Task 3: no Secrets

Find where the operator creates or expects `bootstrap-hub-kubeconfig` and `hub-kubeconfig-secret`
(and the external-managed variants in hosted mode). In `spiffe` mode:

- don't wait for `bootstrap-hub-kubeconfig`;
- don't create `hub-kubeconfig-secret`;
- don't report "HubConnectionDegraded: bootstrap secret missing";
- in hosted modes, keep `external-managed-kubeconfig` handling as it is (Open Question 5).

Switching an existing Klusterlet to `spiffe` deletes the old `hub-kubeconfig-secret` after the
agents are up (proposal, Upgrade). Implement that or leave a TODO linked to Day 25.

## Task 4: tests

Add a table to the existing manifest tests (look for tests that render the Deployments per mode).

| Klusterlet input | Expected output |
| --- | --- |
| `Default`, `spiffe` | two Deployments; labels `spiffe-role: registration` / `work`; CSI volume; SPIFFE args; no `hub-kubeconfig-secret` volume |
| `Singleton`, `spiffe` | one Deployment `klusterlet-agent`, label `klusterlet`, SA `klusterlet-work-sa` |
| `Hosted`, `spiffe`, Klusterlet `cluster4` | Deployments in namespace `cluster4`, SAs `cluster4-registration-sa` / `cluster4-work-sa` |
| `SingletonHosted`, `spiffe`, `cluster4` | `cluster4-agent`, label `klusterlet`, SA `cluster4-work-sa` |
| any mode, `spiffe` | no create of `hub-kubeconfig-secret`; not degraded without a bootstrap Secret |
| `Default`, `csr` | unchanged (no label, no CSI volume) |

Example check:

```go
require.Equal(t, "work", deploy.Spec.Template.Labels["open-cluster-management.io/spiffe-role"])
require.Equal(t, "klusterlet-work-sa", deploy.Spec.Template.Spec.ServiceAccountName)
require.True(t, hasCSIVolume(deploy, "csi.spiffe.io"))
require.False(t, hasVolume(deploy, "hub-kubeconfig-secret"))
```

**Run:**

```shell
dgo 'go test ./pkg/operator/operators/klusterlet/... -run SPIFFE -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'make test-registration-operator-integration' 2>&1 | tail -2
```

## Task 5: commit

```shell
git add -A && git commit -s -m ":sparkles: render SPIFFE agents for all four Klusterlet modes"
```

## Example: a SPIRE entry that matches what the operator renders

For the platform team (from the proposal), cluster1's work agent in `Default` mode:

```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: ocm-work-agent
spec:
  spiffeIDTemplate: "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work"
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: open-cluster-management-agent
  podSelector:
    matchLabels:
      open-cluster-management.io/spiffe-role: work
  workloadSelectorTemplates:
  - "k8s:sa:klusterlet-work-sa"
```

## Rules check

- [ ] No bootstrap or hub-kubeconfig Secret in any of the four modes.
- [ ] Every agent pod has the role label that matches its ServiceAccount.

## Update: multiple agent trust domains

Render `--spiffe-trust-domain={{ .SPIFFETrustDomain }}` for every agent. In hosted modes it is the management cluster's trust domain, because the agents get their SVIDs there.
