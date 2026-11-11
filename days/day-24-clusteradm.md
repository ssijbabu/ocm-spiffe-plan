# Day 24: clusteradm

**Goal:** `clusteradm init`, `join` and `accept` support `spiffe` in all four modes, with no
token.

**Repo:** `clusteradm`, branch `spire`, `pkg/cmd/init`, `pkg/cmd/join`, `pkg/cmd/accept`.

## Before you start

| Input | Where |
| --- | --- |
| Design | proposal, clusteradm section |
| Existing driver flags | `pkg/cmd/init/options.go` (`--registration-drivers`), `pkg/cmd/join/options.go` (`--registration-auth`) |
| How the Azure driver was added | `azure-registration-driver` branch (PR #587): same pattern |

Target commands:

```shell
# Hub
clusteradm init --registration-drivers spiffe --spiffe-trust-domain ocm-hub.example.org

# Managed cluster: no token
clusteradm join --cluster-name cluster1 --registration-auth spiffe \
  --hub-grpc-endpoint ocm-grpc.hub.internal:443 \
  --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server

# Hosted and singleton use the existing flags
clusteradm join ... --mode hosted --managed-cluster-kubeconfig ./cluster4.kubeconfig
clusteradm join ... --singleton

# Unchanged
clusteradm accept --clusters cluster1
```

## Task 1: point at the forks

```shell
cd /home/ssijbabu/Documents/repos/ocm/clusteradm
dgo 'go mod edit \
  -replace open-cluster-management.io/api=github.com/ssijbabu/api@<api-sha> \
  -replace open-cluster-management.io/ocm=github.com/ssijbabu/ocm@<ocm-sha> \
  && go mod tidy && go mod vendor && go build ./...'
```

clusteradm's pre-commit hook runs `make check`, and make isn't on the host. Run
`dgo 'make check'` first, then commit with `--no-verify`.

## Task 2: init

**Flags:** `--spiffe-trust-domain` (required with `spiffe`), `--spiffe-server-id` (optional),
`--spiffe-endpoint-type` (`hostname`, `loadBalancer`, `route`; default `loadBalancer`).

**Output:** the ClusterManager it applies.

```yaml
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

**Printed join command:** with `spiffe`, `init` doesn't print a token. It prints:

```text
Your hub is ready for SPIFFE registration. On each managed cluster, after its SPIRE entries exist, run:

    clusteradm join --cluster-name <cluster_name> --registration-auth spiffe \
      --hub-grpc-endpoint <grpc-spiffe address> \
      --hub-spiffe-id spiffe://ocm-hub.example.org/ocm/hub/grpc-server
```

Get the address from the `cluster-manager-grpc-server-spiffe` Service once its load balancer has an
address. If it has none yet, print a placeholder and the `kubectl` command to read it.

## Task 3: join

**Flags:** `--registration-auth spiffe`, `--hub-grpc-endpoint`, `--hub-spiffe-id`,
`--spiffe-workload-api-socket` (optional). With `spiffe`, `--hub-token` and `--hub-apiserver` are
not needed; refuse them with a clear error so nobody passes a token by mistake.

**Output:** the Klusterlet it applies.

```yaml
spec:
  clusterName: cluster1
  deployOption:
    mode: Default          # Singleton with --singleton; Hosted with --mode hosted; SingletonHosted with both
  registrationConfiguration:
    featureGates:
    - feature: SPIFFERegistration
      mode: Enable
    registrationDriver:
      authType: spiffe
      spiffe:
        hubEndpoint: ocm-grpc.hub.internal:443
        hubSPIFFEID: spiffe://ocm-hub.example.org/ocm/hub/grpc-server
    addOnKubeClientRegistrationDriver:
      authType: spiffe
```

No bootstrap Secret is created. After applying, `join` waits for `SPIFFEAdmissionReady=True` on the
Klusterlet and prints:

```text
Waiting for the Klusterlet to secure its namespaces (SPIFFEAdmissionReady)... done.
Create the SPIRE entries for cluster1 now, then accept it on the hub:

    clusteradm accept --clusters cluster1
```

With `--wait=false`, it prints that the platform must wait for the condition itself.

## Task 4: accept

With `spiffe`, there's no CSR. `accept` sets `hubAcceptsClient: true` on the ManagedCluster and
doesn't wait for a CSR (the Azure driver change did the same; reuse it).

## Task 5: tests

| Command input | Expected |
| --- | --- |
| `init --registration-drivers spiffe` without trust domain | error: `--spiffe-trust-domain is required with spiffe` |
| `init --registration-drivers spiffe --spiffe-trust-domain ocm-hub.example.org` | ClusterManager as in Task 2 |
| `join --registration-auth spiffe` without `--hub-spiffe-id` | error naming the flag |
| `join --registration-auth spiffe --hub-token x ...` | error: tokens are not used with spiffe |
| `join` spiffe, `--singleton` | `mode: Singleton` |
| `join` spiffe, `--mode hosted --managed-cluster-kubeconfig k` | `mode: Hosted` |
| `join` spiffe, `--mode hosted --singleton` | `mode: SingletonHosted` |
| `join` spiffe, `--dry-run` | prints the Klusterlet; no Secret in the output |
| `accept` for a spiffe cluster | no CSR wait; `hubAcceptsClient: true` |

**Run:**

```shell
dgo 'go test ./pkg/cmd/init/... ./pkg/cmd/join/... ./pkg/cmd/accept/... -v -run SPIFFE' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'make test' 2>&1 | tail -3
```

## Task 6: commit and draft PR

```shell
dgo 'make check' && git add -A && git commit -s --no-verify -m ":sparkles: support the spiffe registration driver in init, join and accept"
git push fork spire
gh pr create -R open-cluster-management-io/clusteradm --draft --head ssijbabu:spire \
  --title ":sparkles: Support SPIFFE registration" --body "Part of enhancement #199."
```

## Rules check

- [ ] No token or bootstrap Secret is created or printed.
- [ ] All four modes work.

## Update: multiple agent trust domains

`init` gets `--spiffe-agent-trust-domains` (sets `agentTrustDomains`), `join` gets `--spiffe-trust-domain` (sets the Klusterlet's `trustDomain`), and `accept` gets `--spiffe-trust-domain`, which refuses to accept a cluster whose annotation differs, and sets it when it is missing.
