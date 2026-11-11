# Day 4: SPIFFE ID parsing and mapping

**Goal:** a tested function that turns an agent's SPIFFE ID into the OCM user and groups from the
proposal's ID-scheme table, and rejects everything else.

**Repo:** `sdk-go`, branch `spire`, package `pkg/server/grpc/authn`.

## Before you start

| Input | Where |
| --- | --- |
| ID scheme | proposal, "SPIFFE ID scheme" table (frozen part) |
| Existing authenticators | `pkg/server/grpc/authn/mtls.go`, `token.go`, `interface.go` |

The mapping to implement (from the proposal, `<td>` = trust domain):

| SPIFFE ID | User | Groups |
| --- | --- | --- |
| `spiffe://<td>/ocm/cluster/<c>/agent/registration` | `system:open-cluster-management:<c>:registration` | `system:open-cluster-management:<c>`, `system:open-cluster-management:managed-clusters` |
| `spiffe://<td>/ocm/cluster/<c>/agent/work` | `system:open-cluster-management:<c>:work` | same |
| `spiffe://<td>/ocm/cluster/<c>/agent/klusterlet` | `system:open-cluster-management:<c>:klusterlet` | same |
| `spiffe://<td>/ocm/cluster/<c>/addon/<a>/agent/<g>` | `system:open-cluster-management:cluster:<c>:addon:<a>:agent:<g>` | `system:open-cluster-management:cluster:<c>:addon:<a>`, `system:open-cluster-management:addon:<a>` |
| `spiffe://<td>/ocm/hub/grpc-server` | n/a, the server; refused as a client | |

Every mapped identity also gets `system:authenticated`.

## Task 1: add go-spiffe

```shell
cd /home/ssijbabu/Documents/repos/ocm/sdk-go
dgo 'go get github.com/spiffe/go-spiffe/v2@latest && go mod tidy && go mod vendor'
grep go-spiffe go.mod
```

**Output:**

```text
	github.com/spiffe/go-spiffe/v2 v2.6.0
```

(Use whatever the latest released version is; check `vendor/` is used by sdk-go first. If it
isn't, skip `go mod vendor`.)

## Task 2: write the parser

**New file:** `pkg/server/grpc/authn/spiffeid.go`

```go
package authn

import (
	"fmt"
	"strings"

	"github.com/spiffe/go-spiffe/v2/spiffeid"
	"k8s.io/apimachinery/pkg/util/validation"
)

// AgentRole is the role of a Klusterlet agent.
type AgentRole string

const (
	RoleRegistration AgentRole = "registration"
	RoleWork         AgentRole = "work"
	RoleKlusterlet   AgentRole = "klusterlet"
)

// OCMIdentity is what an agent SPIFFE ID means to OCM.
type OCMIdentity struct {
	ClusterName string
	Role        AgentRole // set for Klusterlet agents
	AddOnName   string    // set for add-on agents
	AgentName   string    // set for add-on agents
	User        string
	Groups      []string
}

// IsAddOn reports whether the identity is an add-on agent.
func (i OCMIdentity) IsAddOn() bool { return i.AddOnName != "" }

// ParseAgentID maps an agent SPIFFE ID in trust domain td to an OCM identity.
func ParseAgentID(id spiffeid.ID, td spiffeid.TrustDomain) (OCMIdentity, error) {
	if id.TrustDomain() != td {
		return OCMIdentity{}, fmt.Errorf("SPIFFE ID %q is not in trust domain %q", id, td)
	}
	segs := strings.Split(strings.TrimPrefix(id.Path(), "/"), "/")
	switch {
	case len(segs) == 5 && segs[0] == "ocm" && segs[1] == "cluster" && segs[3] == "agent":
		cluster, role := segs[2], AgentRole(segs[4])
		if err := validName("cluster", cluster); err != nil {
			return OCMIdentity{}, err
		}
		switch role {
		case RoleRegistration, RoleWork, RoleKlusterlet:
		default:
			return OCMIdentity{}, fmt.Errorf("unknown agent role %q in %q", role, id)
		}
		return OCMIdentity{
			ClusterName: cluster,
			Role:        role,
			User:        fmt.Sprintf("system:open-cluster-management:%s:%s", cluster, role),
			Groups: []string{
				"system:open-cluster-management:" + cluster,
				"system:open-cluster-management:managed-clusters",
				"system:authenticated",
			},
		}, nil
	case len(segs) == 7 && segs[0] == "ocm" && segs[1] == "cluster" && segs[3] == "addon" && segs[5] == "agent":
		cluster, addon, agent := segs[2], segs[4], segs[6]
		for k, v := range map[string]string{"cluster": cluster, "add-on": addon, "agent": agent} {
			if err := validName(k, v); err != nil {
				return OCMIdentity{}, err
			}
		}
		return OCMIdentity{
			ClusterName: cluster,
			AddOnName:   addon,
			AgentName:   agent,
			User:        fmt.Sprintf("system:open-cluster-management:cluster:%s:addon:%s:agent:%s", cluster, addon, agent),
			Groups: []string{
				fmt.Sprintf("system:open-cluster-management:cluster:%s:addon:%s", cluster, addon),
				"system:open-cluster-management:addon:" + addon,
				"system:authenticated",
			},
		}, nil
	}
	return OCMIdentity{}, fmt.Errorf("SPIFFE ID %q does not follow the OCM agent ID scheme", id)
}

func validName(kind, v string) error {
	if errs := validation.IsDNS1123Label(v); len(errs) > 0 {
		return fmt.Errorf("invalid %s name %q: %s", kind, v, strings.Join(errs, "; "))
	}
	return nil
}
```

Notes:

- `spiffeid.ID` already rejects empty segments, `..`, query strings and upper-case trust domains,
  so the parser only checks the OCM shape.
- The hub's own ID (`/ocm/hub/grpc-server`) falls through to the error, so an agent can't present
  it.
- Check that cluster names really are DNS-1123 labels in OCM. If they can be subdomains (with
  dots), use `IsDNS1123Subdomain` for the cluster.

## Task 3: table tests

**New file:** `pkg/server/grpc/authn/spiffeid_test.go`

```go
func TestParseAgentID(t *testing.T) {
	td := spiffeid.RequireTrustDomainFromString("ocm-hub.example.org")
	cases := []struct {
		name     string
		id       string
		wantUser string
		wantErr  string
	}{
		{"registration", "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/registration",
			"system:open-cluster-management:cluster1:registration", ""},
		{"work", "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work",
			"system:open-cluster-management:cluster1:work", ""},
		{"klusterlet", "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/klusterlet",
			"system:open-cluster-management:cluster1:klusterlet", ""},
		{"addon", "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/addon/my-addon/agent/my-addon-agent",
			"system:open-cluster-management:cluster:cluster1:addon:my-addon:agent:my-addon-agent", ""},
		{"other trust domain", "spiffe://other.example.org/ocm/cluster/cluster1/agent/work", "", "not in trust domain"},
		{"hub id", "spiffe://ocm-hub.example.org/ocm/hub/grpc-server", "", "does not follow"},
		{"unknown role", "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/admin", "", "unknown agent role"},
		{"something else", "spiffe://ocm-hub.example.org/ocm/something-else", "", "does not follow"},
		{"extra segment", "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work/x", "", "does not follow"},
		{"bad cluster name", "spiffe://ocm-hub.example.org/ocm/cluster/Cluster_1/agent/work", "", "invalid cluster name"},
	}
	for _, c := range cases {
		t.Run(c.name, func(t *testing.T) {
			got, err := ParseAgentID(spiffeid.RequireFromString(c.id), td)
			if c.wantErr != "" {
				if err == nil || !strings.Contains(err.Error(), c.wantErr) {
					t.Fatalf("want error containing %q, got %v", c.wantErr, err)
				}
				return
			}
			if err != nil {
				t.Fatal(err)
			}
			if got.User != c.wantUser {
				t.Errorf("user: want %q, got %q", c.wantUser, got.User)
			}
			if !slices.Contains(got.Groups, "system:authenticated") {
				t.Errorf("groups %v miss system:authenticated", got.Groups)
			}
		})
	}
}
```

Also assert the full group list for one Klusterlet case and one add-on case, so a wrong group
string fails the test.

**Run:**

```shell
dgo 'go test ./pkg/server/grpc/authn/ -run TestParseAgentID -v' 2>&1 | tail -15
```

**Output:**

```text
=== RUN   TestParseAgentID/registration
...
--- PASS: TestParseAgentID (0.00s)
    --- PASS: TestParseAgentID/registration (0.00s)
    --- PASS: TestParseAgentID/work (0.00s)
    ...
    --- PASS: TestParseAgentID/bad_cluster_name (0.00s)
PASS
ok  	open-cluster-management.io/sdk-go/pkg/server/grpc/authn	0.012s
```

## Task 4: commit

```shell
git add -A && git commit -s -m ":sparkles: map SPIFFE IDs to OCM identities

Parse spiffe://<td>/ocm/cluster/<c>/agent/<role> and
spiffe://<td>/ocm/cluster/<c>/addon/<a>/agent/<g> into the OCM user and groups
from enhancement #199, plus system:authenticated. Reject other trust domains
and any other path."
```

## End of day

| Output | Check |
| --- | --- |
| `spiffeid.go`, `spiffeid_test.go` | tests pass |
| `go-spiffe/v2` in `go.mod` | `go build ./...` passes |

## Rules check

- [ ] Mapping matches the proposal's table exactly, including `system:authenticated`.
- [ ] The hub's own ID can't be used as a client identity.

## Update: multiple agent trust domains

`ParseAgentID` takes the set of accepted trust domains (`agentTrustDomains`) instead of one, and returns the SVID's trust domain in `OCMIdentity.TrustDomain`. Test rows: an ID from `east.example.org` is accepted when it is in the set; `south.example.org` is refused with `not in agentTrustDomains`.
