# Day 14: the cluster-manager operator renders the SPIFFE hub

**Goal:** with `authType: spiffe` on the ClusterManager, the operator deploys the gRPC server with
the Workload API volume and the SPIFFE listener, adds the `cluster-manager-grpc-server-spiffe`
Service, and creates no signer, serving certificate or bootstrap token when `spiffe` is the only
driver.

**Repo:** `ocm`, branch `spire`, packages `pkg/operator/operators/clustermanager` and
`manifests/cluster-manager`.

## Before you start

| Input | Where |
| --- | --- |
| gRPC server Deployment template | `manifests/cluster-manager/management/grpc-server/deployment.yaml` |
| gRPC server Service template | `manifests/cluster-manager/hub/grpc-server/service.yaml` (port 8090; 443 for `loadBalancer`) |
| Config struct | `manifests/config.go`: `HubConfig` |
| Where drivers are read | `clustermanager_runtime_reconcile.go` (`case operatorapiv1.GRPCAuthType`), `clustermanager_controller.go` (`config.GRPCAuthEnabled`) |

Today, with `grpc`, the Deployment mounts `grpc-server-config` (ConfigMap), `ca-bundle-configmap`
and the Secret `grpc-server-serving-cert`.

## Task 1: config fields

**Change** `manifests/config.go`:

```go
type HubConfig struct {
	// ... existing ...
	SPIFFEAuthEnabled       bool
	GRPCOnlySPIFFE          bool   // spiffe is the only gRPC-based driver: no grpc listener, no certs
	SPIFFETrustDomain       string
	SPIFFEServerID          string
	SPIFFEWorkloadAPISocket string
	SPIFFEMaxSVIDLifetime   string // "24h0m0s", passed to the registration hub controller
	SPIFFEMaxClockSkew      string
	SPIFFEEndpointType      string // hostname | loadBalancer | route, from the grpc-spiffe exposure
}
```

Fill them in the controller from the `spiffe` driver and the `grpc-spiffe` endpoint exposure. Add a
`helpers.SPIFFEAuthEnabled(cm)` next to `helpers.GRPCAuthEnabled(cm)`, and make the gRPC server
Deployment render when either is true.

Also validate here what the CRD can't (see `plan/questions.md`, Q4): if `serverSPIFFEID` is set
and not in `trustDomain`, render nothing for SPIFFE and set `Degraded=True` with reason
`InvalidSPIFFEConfiguration` and message `serverSPIFFEID <id> is not in trust domain <td>`. Parse
both with go-spiffe (`spiffeid.FromString`, then compare `TrustDomain()`), not with string
prefixes, so `spiffe://ocm-hub.example.org.evil/...` is refused. Add both cases to the Task 5
table.

## Task 2: the Deployment

**Change** `deployment.yaml` (only the parts that differ):

```yaml
        args:
          - "/server"
          - "grpc"
          {{ if not .GRPCOnlySPIFFE }}
          - "--server-config=/var/run/secrets/hub/grpc/config/config.yaml"
          {{ else }}
          - "--grpc-enabled=false"
          {{ end }}
          {{ if .SPIFFEAuthEnabled }}
          - "--spiffe-enabled"
          - "--spiffe-listen-address=:8091"
          - "--spiffe-trust-domain={{ .SPIFFETrustDomain }}"
          - "--spiffe-server-id={{ .SPIFFEServerID }}"
          - "--spiffe-workload-api-socket={{ .SPIFFEWorkloadAPISocket }}"
          {{ end }}
        volumeMounts:
        - name: tmpdir
          mountPath: /tmp
        {{ if not .GRPCOnlySPIFFE }}
        - name: grpc-server-config     # unchanged
        - name: grpc-server-ca
        - name: grpc-server-serving-cert
        {{ end }}
        {{ if .SPIFFEAuthEnabled }}
        - name: spiffe-workload-api
          mountPath: /spiffe-workload-api
          readOnly: true
        {{ end }}
      volumes:
      {{ if .SPIFFEAuthEnabled }}
      - name: spiffe-workload-api
        csi:
          driver: csi.spiffe.io
          readOnly: true
      {{ end }}
```

`SPIFFEServerID` is computed by the operator when the field is empty:
`spiffe://<trustDomain>/ocm/hub/grpc-server`. The health probe on 8443 stays.

The pod keeps the label `app: cluster-manager-grpc-server` and ServiceAccount `grpc-server-sa`.
The platform's `ClusterSPIFFEID` for the server selects on those (proposal, hub setup).

## Task 3: the Service

**New file:** `manifests/cluster-manager/hub/grpc-server/service-spiffe.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .ClusterManagerName }}-grpc-server-spiffe
  namespace: {{ .ClusterManagerNamespace }}
spec:
  selector:
    app: {{ .ClusterManagerName }}-grpc-server
  ports:
    - protocol: TCP
  {{ if eq .SPIFFEEndpointType "loadBalancer" }}
      port: 443
  {{ else }}
      port: 8091
  {{ end }}
      targetPort: 8091
  {{ if eq .SPIFFEEndpointType "loadBalancer" }}
  type: LoadBalancer
  {{ else }}
  type: ClusterIP
  {{ end }}
```

Add it to the list of hub resources applied when `SPIFFEAuthEnabled`, and to the list deleted when
it's turned off.

## Task 4: skip Secrets in SPIFFE-only mode

Find where `grpc-server-serving-cert`, the gRPC signer Secret and the bootstrap token are created
(cert rotation controller, `GRPCAuthEnabled` branches in `clustermanager_hub_reconcile.go` and
`clustermanager_runtime_reconcile.go`). Gate them on `GRPCAuthEnabled` only, never on
`SPIFFEAuthEnabled`. When `grpc` is removed from the drivers, the existing cleanup deletes them;
check it does.

Also pass the delete-guard settings to the registration hub controller Deployment:

```yaml
          {{ if .SPIFFEAuthEnabled }}
          - "--spiffe-max-svid-lifetime={{ .SPIFFEMaxSVIDLifetime }}"
          - "--spiffe-max-clock-skew={{ .SPIFFEMaxClockSkew }}"
          {{ end }}
```

## Task 5: tests

Add cases to the existing manifest tests (look for the table that renders the gRPC server
Deployment with `grpc`).

| ClusterManager input | Expected rendered output |
| --- | --- |
| `spiffe` only, trust domain `ocm-hub.example.org` | Deployment args contain `--grpc-enabled=false`, `--spiffe-enabled`, `--spiffe-server-id=spiffe://ocm-hub.example.org/ocm/hub/grpc-server`; volume `csi.spiffe.io`; **no** `grpc-server-serving-cert` volume; Service `cluster-manager-grpc-server-spiffe` on 8091 |
| `spiffe` only | No Secret `grpc-server-serving-cert`, no signer Secret, no bootstrap token Secret in the hub namespace |
| `spiffe` + `grpc` | both listeners' args; both Services; the serving cert exists (for `grpc`) |
| `spiffe` with `grpc-spiffe` exposure `loadBalancer` | Service type `LoadBalancer`, port 443 |
| `serverSPIFFEID` set | args use it unchanged |
| `grpc` removed from `spiffe` + `grpc` | `cluster-manager-grpc-server` Service and its Secrets deleted |

Example assertion:

```go
for _, v := range deploy.Spec.Template.Spec.Volumes {
	require.NotEqual(t, "grpc-server-serving-cert", v.Name, "spiffe-only must not mount a serving cert")
}
```

**Run:**

```shell
dgo 'go test ./pkg/operator/operators/clustermanager/... -v -run "SPIFFE|Spiffe"' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'make test-unit' 2>&1 | grep -E '^(FAIL|--- FAIL)'      # nothing new
```

## Task 6: commit

```shell
git add -A && git commit -s -m ":sparkles: render the SPIFFE gRPC server from the ClusterManager"
```

## Example: what the admin sees

```shell
kubectl -n open-cluster-management-hub get deploy,svc,secret | grep grpc
```

```text
deployment.apps/cluster-manager-grpc-server   1/1   1   1   2m
service/cluster-manager-grpc-server-spiffe    ClusterIP   10.0.12.34   <none>   8091/TCP   2m
```

No `grpc-server-serving-cert` Secret, no `cluster-manager-grpc-server` Service.

## Rules check

- [ ] SPIFFE-only creates no Secret or ConfigMap with keys, certificates or tokens.
- [ ] Hosted cluster-manager mode still mounts `grpc-server-sa-kubeconfig` to reach the hub
      kube-apiserver. That's the hub's own API access, not an agent credential; note it in the PR.
