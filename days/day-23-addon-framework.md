# Day 23: addon-framework

**Goal:** add-on agents get the Workload API instead of a hub-kubeconfig Secret, add-on authors get
a helper to build gRPC clients from an SVID, and the add-on namespace is created with the
dedicated-namespace label.

**Repo:** `addon-framework`, branch `spire` (cloned on Day 1).

## Before you start

| Input | Where |
| --- | --- |
| Where the framework adds the hub kubeconfig volume | search for `hub-kubeconfig` in `pkg/addonfactory` and `pkg/utils` (the code that injects the Secret volume into add-on Deployments) |
| How the framework knows the cluster's add-on driver | the `ManagedClusterAddOn` status `registrations[].kubeClient.driver` (Day 22 sets `spiffe`) |
| Client helper | sdk-go `NewSPIFFEGRPCOptions` (Day 7) |

## Task 1: point at the forks

```shell
cd /home/ssijbabu/Documents/repos/ocm/addon-framework
dgo 'go mod edit \
  -replace open-cluster-management.io/api=github.com/ssijbabu/api@<api-sha> \
  -replace open-cluster-management.io/sdk-go=github.com/ssijbabu/sdk-go@<sdk-go-sha> \
  && go mod tidy && (test -d vendor && go mod vendor || true) && go build ./...'
git commit -s -am ":seedling: point api and sdk-go at the spiffe forks"
```

## Task 2: render add-on workloads for SPIFFE

When the add-on's `kubeClient.driver` is `spiffe`, for every Deployment, DaemonSet and StatefulSet
the framework renders:

- remove the `<addon>-hub-kubeconfig` Secret volume and its mount;
- add the CSI volume and mount;
- add the label `open-cluster-management.io/spiffe-role: addon` to the pod template.

```go
func applySPIFFE(podSpec *corev1.PodSpec, podLabels map[string]string, addonName string) {
	secretVol := addonName + "-hub-kubeconfig"
	podSpec.Volumes = slices.DeleteFunc(podSpec.Volumes, func(v corev1.Volume) bool {
		return v.Secret != nil && v.Secret.SecretName == secretVol
	})
	for i := range podSpec.Containers {
		c := &podSpec.Containers[i]
		c.VolumeMounts = slices.DeleteFunc(c.VolumeMounts, func(m corev1.VolumeMount) bool { return m.Name == "hub-kubeconfig" })
		c.VolumeMounts = append(c.VolumeMounts, corev1.VolumeMount{Name: "spiffe-workload-api", MountPath: "/spiffe-workload-api", ReadOnly: true})
	}
	ro := true
	podSpec.Volumes = append(podSpec.Volumes, corev1.Volume{
		Name:         "spiffe-workload-api",
		VolumeSource: corev1.VolumeSource{CSI: &corev1.CSIVolumeSource{Driver: "csi.spiffe.io", ReadOnly: &ro}},
	})
	podLabels["open-cluster-management.io/spiffe-role"] = "addon"
}
```

Use the real volume and mount names the framework uses today (check them; `hub-kubeconfig` above
is a guess).

**Input / output example.** Input Deployment from an add-on's manifests:

```yaml
spec:
  template:
    metadata:
      labels: {app: my-addon-agent}
    spec:
      serviceAccountName: my-addon-agent-sa
      containers:
      - name: agent
        volumeMounts:
        - {name: hub-kubeconfig, mountPath: /var/run/hub}
      volumes:
      - name: hub-kubeconfig
        secret: {secretName: my-addon-hub-kubeconfig}
```

Rendered for a `spiffe` cluster:

```yaml
spec:
  template:
    metadata:
      labels:
        app: my-addon-agent
        open-cluster-management.io/spiffe-role: addon
    spec:
      serviceAccountName: my-addon-agent-sa
      containers:
      - name: agent
        volumeMounts:
        - {name: spiffe-workload-api, mountPath: /spiffe-workload-api, readOnly: true}
      volumes:
      - name: spiffe-workload-api
        csi: {driver: csi.spiffe.io, readOnly: true}
```

## Task 3: the namespace label

When the framework renders the add-on's install Namespace manifest for a `spiffe` cluster, add:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-addon
  labels:
    open-cluster-management.io/spiffe-addon-namespace: klusterlet   # the Klusterlet's name
```

Find where the framework learns the Klusterlet's name (the cluster's ManagedCluster may not carry
it). If nothing carries it, add it to what the registration agent reports (for example a
ManagedCluster annotation or claim) and note the change in `ocm`.

If the namespace already exists without the label, the work agent's apply is refused by the
namespace-label policy (Day 20), and the registration agent reports `AddOnNamespaceNotDedicated`
(Day 22). The framework doesn't need to handle it.

## Task 4: the client helper

**New file:** `pkg/agent/spiffe.go` (or next to the existing agent helpers)

```go
// SPIFFEClientOptions builds CloudEvents gRPC options for an add-on agent from its SVID.
//
//	opts, closer, err := agent.SPIFFEClientOptions(ctx, agent.SPIFFEAgent{
//		ClusterName: "cluster1", AddOnName: "my-addon", AgentName: "my-addon-agent",
//		HubEndpoint: "ocm-grpc.hub.internal:443",
//		HubSPIFFEID: "spiffe://ocm-hub.example.org/ocm/hub/grpc-server",
//	})
func SPIFFEClientOptions(ctx context.Context, a SPIFFEAgent) (*grpcoptions.GRPCOptions, io.Closer, error) {
	td := spiffeid.RequireFromString(a.HubSPIFFEID).TrustDomain().Name()
	return grpcoptions.NewSPIFFEGRPCOptions(ctx, grpcoptions.SPIFFEConfig{
		URL:               a.HubEndpoint,
		HubSPIFFEID:       a.HubSPIFFEID,
		AgentSPIFFEID:     fmt.Sprintf("spiffe://%s/ocm/cluster/%s/addon/%s/agent/%s", td, a.ClusterName, a.AddOnName, a.AgentName),
		WorkloadAPISocket: cmp.Or(a.WorkloadAPISocket, "unix:///spiffe-workload-api/spire-agent.sock"),
	})
}
```

How does the add-on agent learn `HubEndpoint` and `HubSPIFFEID`? They aren't secret. Pass them
through the add-on's rendered environment or flags (the framework already passes cluster name and
add-on name). Decide and write it in the PR.

## Task 5: tests

| Case | Input | Expected |
| --- | --- | --- |
| Deployment, spiffe | the input above | output above |
| DaemonSet, spiffe | same pod spec in a DaemonSet | same changes |
| StatefulSet, spiffe | same pod spec in a StatefulSet | same changes |
| csr driver | any | unchanged, Secret volume kept |
| Namespace, spiffe | install namespace `my-addon` | label set to the Klusterlet name |
| Helper | cluster1, my-addon, my-addon-agent | agent ID `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/addon/my-addon/agent/my-addon-agent` |

**Run:**

```shell
dgo 'go test ./pkg/... -run SPIFFE -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'make test' 2>&1 | tail -3
```

## Task 6: commit and draft PR

```shell
git add -A && git commit -s -m ":sparkles: render add-on agents for the spiffe driver"
git push fork spire
gh pr create -R open-cluster-management-io/addon-framework --draft --head ssijbabu:spire \
  --title ":sparkles: SPIFFE add-on agents" --body "Part of enhancement #199."
```

## Rules check

- [ ] Add-on agents get no Secret; each has its own SVID.
- [ ] The add-on namespace is labeled at creation, never later.
