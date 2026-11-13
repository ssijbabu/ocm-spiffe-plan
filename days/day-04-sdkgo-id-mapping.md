# Day 4: SPIFFE ID templates and mapping

**Goal:** a tested package that compiles, renders and matches agent ID templates, and a tested
`IDScheme` that turns an agent's SPIFFE ID into the OCM user and groups, and rejects everything
else.

**Repo:** `sdk-go`, branch `spire`. New package `pkg/spiffe/idtemplate`; the mapping goes in
`pkg/server/grpc/authn`.

## Before you start

| Input | Where |
| --- | --- |
| ID scheme | proposal, "SPIFFE ID scheme" table (frozen part): it is the default template |
| Templates | `questions.md` Q7; api `DefaultSPIFFEAgentIDTemplate`, `DefaultSPIFFEAddOnAgentIDTemplate` (Day 2 update) |
| Existing authenticators | `pkg/server/grpc/authn/mtls.go`, `token.go`, `interface.go` |

The hub reads the cluster, the role, the add-on and the agent name from the ID, so the ID layout
is described by two templates:

| Template | Default | Placeholders |
| --- | --- | --- |
| `agentIDTemplate` | `spiffe://{trustDomain}/ocm/cluster/{cluster}/agent/{role}` | `{cluster}`, `{role}` required; `{namespace}` optional |
| `addOnAgentIDTemplate` | `spiffe://{trustDomain}/ocm/cluster/{cluster}/addon/{addon}/agent/{agent}` | `{cluster}`, `{addon}`, `{agent}` required; `{namespace}` optional |

Template rules:

- It starts with `spiffe://{trustDomain}/`. The trust domain is checked against
  `agentTrustDomains`, never taken from the path.
- Each placeholder fills a whole path segment (`/{cluster}/`, not `/cluster-{cluster}/`) and
  appears at most once.
- At least one segment is fixed text, so a template can't match every ID in a trust domain.
- No SPIFFE ID may match both templates, and the hub's own ID must match neither.
- `{namespace}` is checked to be a DNS-1123 label and returned for logs, but never used for
  authorization: the hub can't verify it.

The mapping (from the proposal, `<c>` cluster, `<a>` add-on, `<g>` add-on agent):

| Template match | User | Groups |
| --- | --- | --- |
| agent, `{role}` = `registration` | `system:open-cluster-management:<c>:registration` | `system:open-cluster-management:<c>`, `system:open-cluster-management:managed-clusters` |
| agent, `{role}` = `work` | `system:open-cluster-management:<c>:work` | same |
| agent, `{role}` = `klusterlet` | `system:open-cluster-management:<c>:klusterlet` | same |
| add-on agent | `system:open-cluster-management:cluster:<c>:addon:<a>:agent:<g>` | `system:open-cluster-management:cluster:<c>:addon:<a>`, `system:open-cluster-management:addon:<a>` |
| the hub's `serverSPIFFEID` | n/a, the server; refused as a client | |

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
isn't, skip `go mod vendor`.) Point the `api` `replace` line at the commit with the Day 2/3
template update.

## Task 2: the template package

**New file:** `pkg/spiffe/idtemplate/idtemplate.go`. It has no OCM identity logic, so the hub,
the Klusterlet operator, the add-on framework and clusteradm can all use it.

```go
// Package idtemplate compiles, renders and matches OCM agent SPIFFE ID templates.
package idtemplate

import (
	"fmt"
	"slices"
	"strings"

	"github.com/spiffe/go-spiffe/v2/spiffeid"
)

// Kind says which agents a template is for. Its value is the API field name, for messages.
type Kind string

const (
	KindAgent      Kind = "agentIDTemplate"
	KindAddOnAgent Kind = "addOnAgentIDTemplate"
)

// Placeholder names.
const (
	Cluster   = "cluster"
	Role      = "role"
	AddOn     = "addon"
	Agent     = "agent"
	Namespace = "namespace"
)

const prefix = "spiffe://{trustDomain}/"

var (
	allowed = map[Kind][]string{
		KindAgent:      {Cluster, Role, Namespace},
		KindAddOnAgent: {Cluster, AddOn, Agent, Namespace},
	}
	required = map[Kind][]string{
		KindAgent:      {Cluster, Role},
		KindAddOnAgent: {Cluster, AddOn, Agent},
	}
)

type segment struct {
	literal     string // fixed text
	placeholder string // name of a {placeholder}
}

// Template is a checked agent ID template.
type Template struct {
	kind Kind
	raw  string
	segs []segment
}

// Compile parses s and checks it against the rules for kind.
func Compile(kind Kind, s string) (*Template, error) {
	if !strings.HasPrefix(s, prefix) {
		return nil, fmt.Errorf("%s %q must start with %s", kind, s, prefix)
	}
	t := &Template{kind: kind, raw: s}
	seen := map[string]bool{}
	fixed := 0
	for _, p := range strings.Split(strings.TrimPrefix(s, prefix), "/") {
		switch {
		case strings.HasPrefix(p, "{") && strings.HasSuffix(p, "}"):
			name := p[1 : len(p)-1]
			if !slices.Contains(allowed[kind], name) {
				return nil, fmt.Errorf("%s %q: unknown placeholder {%s}, allowed: %s",
					kind, s, name, strings.Join(allowed[kind], ", "))
			}
			if seen[name] {
				return nil, fmt.Errorf("%s %q: {%s} appears more than once", kind, s, name)
			}
			seen[name] = true
			t.segs = append(t.segs, segment{placeholder: name})
		case strings.ContainsAny(p, "{}"):
			return nil, fmt.Errorf("%s %q: a placeholder must fill a whole path segment, got %q", kind, s, p)
		default:
			if err := spiffeid.ValidatePathSegment(p); err != nil {
				return nil, fmt.Errorf("%s %q: segment %q: %w", kind, s, p, err)
			}
			fixed++
			t.segs = append(t.segs, segment{literal: p})
		}
	}
	for _, r := range required[kind] {
		if !seen[r] {
			return nil, fmt.Errorf("%s %q must contain {%s}", kind, s, r)
		}
	}
	if fixed == 0 {
		return nil, fmt.Errorf("%s %q needs at least one fixed path segment", kind, s)
	}
	return t, nil
}

func (t *Template) String() string { return t.raw }

// Has reports whether the template uses the placeholder.
func (t *Template) Has(name string) bool {
	return slices.ContainsFunc(t.segs, func(s segment) bool { return s.placeholder == name })
}

// Match returns the placeholder values when the path of id fits the template. It checks
// neither the trust domain nor the values.
func (t *Template) Match(id spiffeid.ID) (map[string]string, bool) {
	parts := strings.Split(strings.TrimPrefix(id.Path(), "/"), "/")
	if len(parts) != len(t.segs) {
		return nil, false
	}
	values := map[string]string{}
	for i, seg := range t.segs {
		if seg.placeholder == "" {
			if parts[i] != seg.literal {
				return nil, false
			}
			continue
		}
		values[seg.placeholder] = parts[i]
	}
	return values, true
}

// Render fills the template. Every placeholder the template uses needs a value; extra
// values are ignored, so callers can always pass the namespace.
func (t *Template) Render(trustDomain string, values map[string]string) (spiffeid.ID, error) {
	td, err := spiffeid.TrustDomainFromString(trustDomain)
	if err != nil {
		return spiffeid.ID{}, err
	}
	parts := make([]string, 0, len(t.segs))
	for _, seg := range t.segs {
		if seg.placeholder == "" {
			parts = append(parts, seg.literal)
			continue
		}
		v := values[seg.placeholder]
		if v == "" {
			return spiffeid.ID{}, fmt.Errorf("%s %q: no value for {%s}", t.kind, t.raw, seg.placeholder)
		}
		parts = append(parts, v)
	}
	return spiffeid.FromSegments(td, parts...)
}

// Overlaps reports whether some SPIFFE ID could match both templates.
func Overlaps(a, b *Template) bool {
	if len(a.segs) != len(b.segs) {
		return false
	}
	for i := range a.segs {
		x, y := a.segs[i], b.segs[i]
		if x.placeholder == "" && y.placeholder == "" && x.literal != y.literal {
			return false
		}
	}
	return true
}
```

Notes:

- Check the signatures of `spiffeid.ValidatePathSegment` and `spiffeid.FromSegments` in the
  vendored go-spiffe version: both take segments without the leading `/`.
- `spiffeid.ID` already rejects empty segments, `.`/`..`, query strings and upper-case trust
  domains, so `Match` only checks the template's shape.

## Task 3: the mapping

**New file:** `pkg/server/grpc/authn/spiffeid.go`

```go
package authn

import (
	"fmt"
	"slices"
	"strings"

	"github.com/spiffe/go-spiffe/v2/spiffeid"
	"k8s.io/apimachinery/pkg/util/validation"

	"open-cluster-management.io/sdk-go/pkg/spiffe/idtemplate"
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
	TrustDomain string    // the SVID's trust domain, for pinning (Day 11)
	ClusterName string
	Role        AgentRole // set for Klusterlet agents
	AddOnName   string    // set for add-on agents
	AgentName   string    // set for add-on agents
	Namespace   string    // from {namespace} if the template has it; for logs only
	User        string
	Groups      []string
}

// IsAddOn reports whether the identity is an add-on agent.
func (i OCMIdentity) IsAddOn() bool { return i.AddOnName != "" }

// IDScheme maps agent SPIFFE IDs to OCM identities.
type IDScheme struct {
	agent, addOn *idtemplate.Template
	trustDomains []spiffeid.TrustDomain
}

// NewIDScheme checks both templates and the accepted trust domains (agentTrustDomains,
// which always include the hub's own).
func NewIDScheme(agentTemplate, addOnTemplate string, trustDomains []string) (*IDScheme, error) {
	agent, err := idtemplate.Compile(idtemplate.KindAgent, agentTemplate)
	if err != nil {
		return nil, err
	}
	addOn, err := idtemplate.Compile(idtemplate.KindAddOnAgent, addOnTemplate)
	if err != nil {
		return nil, err
	}
	if idtemplate.Overlaps(agent, addOn) {
		return nil, fmt.Errorf("agentIDTemplate %q and addOnAgentIDTemplate %q can match the same SPIFFE ID",
			agent, addOn)
	}
	if len(trustDomains) == 0 {
		return nil, fmt.Errorf("no agent trust domain")
	}
	s := &IDScheme{agent: agent, addOn: addOn}
	for _, name := range trustDomains {
		td, err := spiffeid.TrustDomainFromString(name)
		if err != nil {
			return nil, fmt.Errorf("agent trust domain %q: %w", name, err)
		}
		s.trustDomains = append(s.trustDomains, td)
	}
	return s, nil
}

// CheckServerID refuses a hub server ID that an agent could also present.
func (s *IDScheme) CheckServerID(id spiffeid.ID) error {
	for _, t := range []*idtemplate.Template{s.agent, s.addOn} {
		if _, ok := t.Match(id); ok {
			return fmt.Errorf("serverSPIFFEID %q matches %q; agents could present it", id, t)
		}
	}
	return nil
}

// TrustDomains returns the accepted agent trust domains, for the server TLS authorizer.
func (s *IDScheme) TrustDomains() []spiffeid.TrustDomain { return s.trustDomains }

// Parse maps an agent SPIFFE ID to its OCM identity.
func (s *IDScheme) Parse(id spiffeid.ID) (OCMIdentity, error) {
	if !slices.Contains(s.trustDomains, id.TrustDomain()) {
		return OCMIdentity{}, fmt.Errorf("SPIFFE ID %q: trust domain %q is not in agentTrustDomains",
			id, id.TrustDomain().Name())
	}
	if v, ok := s.agent.Match(id); ok {
		return agentIdentity(id, v)
	}
	if v, ok := s.addOn.Match(id); ok {
		return addOnIdentity(id, v)
	}
	return OCMIdentity{}, fmt.Errorf("SPIFFE ID %q matches neither agentIDTemplate %q nor addOnAgentIDTemplate %q",
		id, s.agent, s.addOn)
}

func agentIdentity(id spiffeid.ID, v map[string]string) (OCMIdentity, error) {
	cluster, role, ns := v[idtemplate.Cluster], AgentRole(v[idtemplate.Role]), v[idtemplate.Namespace]
	if err := validNames(map[string]string{"cluster": cluster, "namespace": ns}); err != nil {
		return OCMIdentity{}, err
	}
	switch role {
	case RoleRegistration, RoleWork, RoleKlusterlet:
	default:
		return OCMIdentity{}, fmt.Errorf("unknown agent role %q in %q", role, id)
	}
	return OCMIdentity{
		TrustDomain: id.TrustDomain().Name(),
		ClusterName: cluster,
		Role:        role,
		Namespace:   ns,
		User:        fmt.Sprintf("system:open-cluster-management:%s:%s", cluster, role),
		Groups: []string{
			"system:open-cluster-management:" + cluster,
			"system:open-cluster-management:managed-clusters",
			"system:authenticated",
		},
	}, nil
}

func addOnIdentity(id spiffeid.ID, v map[string]string) (OCMIdentity, error) {
	cluster, addon, agent, ns := v[idtemplate.Cluster], v[idtemplate.AddOn], v[idtemplate.Agent], v[idtemplate.Namespace]
	if err := validNames(map[string]string{"cluster": cluster, "add-on": addon, "agent": agent, "namespace": ns}); err != nil {
		return OCMIdentity{}, err
	}
	return OCMIdentity{
		TrustDomain: id.TrustDomain().Name(),
		ClusterName: cluster,
		AddOnName:   addon,
		AgentName:   agent,
		Namespace:   ns,
		User:        fmt.Sprintf("system:open-cluster-management:cluster:%s:addon:%s:agent:%s", cluster, addon, agent),
		Groups: []string{
			fmt.Sprintf("system:open-cluster-management:cluster:%s:addon:%s", cluster, addon),
			"system:open-cluster-management:addon:" + addon,
			"system:authenticated",
		},
	}, nil
}

// validNames checks every non-empty value; an empty value is an unused optional placeholder.
func validNames(names map[string]string) error {
	for kind, v := range names {
		if v == "" {
			continue
		}
		if errs := validation.IsDNS1123Label(v); len(errs) > 0 {
			return fmt.Errorf("invalid %s name %q: %s", kind, v, strings.Join(errs, "; "))
		}
	}
	return nil
}
```

Notes:

- The hub's own ID (`/ocm/hub/grpc-server` by default) matches neither default template, so an
  agent can't present it. With custom templates, `CheckServerID` makes sure that stays true; the
  gRPC server (Day 10) and the cluster-manager operator (Day 14) call it.
- Check that cluster names really are DNS-1123 labels in OCM. If they can be subdomains (with
  dots), use `IsDNS1123Subdomain` for the cluster.

## Task 4: tests

**New file:** `pkg/spiffe/idtemplate/idtemplate_test.go`

| Case | Kind | Template | Expected |
| --- | --- | --- | --- |
| Default agent | agent | `spiffe://{trustDomain}/ocm/cluster/{cluster}/agent/{role}` | ok |
| Default add-on | add-on | `spiffe://{trustDomain}/ocm/cluster/{cluster}/addon/{addon}/agent/{agent}` | ok |
| With namespace | agent | `spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}` | ok |
| Wrong prefix | agent | `spiffe://ocm-hub.example.org/ocm/{cluster}/{role}` | `must start with spiffe://{trustDomain}/` |
| Missing `{role}` | agent | `spiffe://{trustDomain}/ocm/cluster/{cluster}` | `must contain {role}` |
| Missing `{agent}` | add-on | `spiffe://{trustDomain}/ocm/{cluster}/{addon}` | `must contain {agent}` |
| Unknown | agent | `spiffe://{trustDomain}/ocm/{clutser}/{role}` | `unknown placeholder {clutser}` |
| Add-on placeholder in agent template | agent | `spiffe://{trustDomain}/ocm/{cluster}/{role}/{addon}` | `unknown placeholder {addon}` |
| Twice | agent | `spiffe://{trustDomain}/ocm/{cluster}/{role}/{cluster}` | `appears more than once` |
| Partial segment | agent | `spiffe://{trustDomain}/ocm/cluster-{cluster}/{role}` | `must fill a whole path segment` |
| No fixed segment | agent | `spiffe://{trustDomain}/{cluster}/{role}` | `needs at least one fixed path segment` |
| Empty segment | agent | `spiffe://{trustDomain}/ocm//{cluster}/{role}` | error from `ValidatePathSegment` |

Also:

- `Render` then `Match` round-trips for both defaults and the namespace template, with values
  `cluster1`, `work`, `open-cluster-management-agent` (and `my-addon`, `my-addon-agent`).
- `Render` without a needed value fails with `no value for {cluster}`; an extra `namespace` value
  with a template that doesn't use it is ignored.
- `Overlaps`: the two defaults don't overlap; `.../ocm/{cluster}/{role}` and
  `.../ocm/{cluster}/{addon}/{agent}` don't (lengths differ); `.../ocm/{cluster}/x/{role}` and
  `.../ocm/{cluster}/{addon}/{agent}` do.

**New file:** `pkg/server/grpc/authn/spiffeid_test.go`. Run the same table against two schemes:
the defaults, and agent template `spiffe://{trustDomain}/k8s/{cluster}/ns/{namespace}/ocm/{role}`
with the default add-on template. Trust domains for both: `ocm-hub.example.org`,
`east.example.org`.

```go
func TestIDSchemeParse(t *testing.T) {
	def, err := NewIDScheme(v1.DefaultSPIFFEAgentIDTemplate, v1.DefaultSPIFFEAddOnAgentIDTemplate,
		[]string{"ocm-hub.example.org", "east.example.org"})
	require.NoError(t, err)
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
		{"listed trust domain", "spiffe://east.example.org/ocm/cluster/cluster2/agent/work",
			"system:open-cluster-management:cluster2:work", ""},
		{"unlisted trust domain", "spiffe://south.example.org/ocm/cluster/cluster4/agent/work", "",
			"not in agentTrustDomains"},
		{"hub id", "spiffe://ocm-hub.example.org/ocm/hub/grpc-server", "", "matches neither"},
		{"unknown role", "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/admin", "", "unknown agent role"},
		{"something else", "spiffe://ocm-hub.example.org/ocm/something-else", "", "matches neither"},
		{"generic ns/sa id", "spiffe://ocm-hub.example.org/ns/open-cluster-management-agent/sa/klusterlet-work-sa", "", "matches neither"},
		{"extra segment", "spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work/x", "", "matches neither"},
		{"bad cluster name", "spiffe://ocm-hub.example.org/ocm/cluster/Cluster_1/agent/work", "", "invalid cluster name"},
	}
	for _, c := range cases {
		t.Run(c.name, func(t *testing.T) {
			got, err := def.Parse(spiffeid.RequireFromString(c.id))
			if c.wantErr != "" {
				require.ErrorContains(t, err, c.wantErr)
				return
			}
			require.NoError(t, err)
			require.Equal(t, c.wantUser, got.User)
			require.Contains(t, got.Groups, "system:authenticated")
		})
	}
}
```

Rows for the namespace scheme:

| ID | Expected |
| --- | --- |
| `spiffe://ocm-hub.example.org/k8s/cluster1/ns/team-a-ocm/ocm/work` | user `system:open-cluster-management:cluster1:work`, `Namespace` `team-a-ocm` |
| `spiffe://ocm-hub.example.org/k8s/cluster1/ns/Team_A/ocm/work` | `invalid namespace name` |
| `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/agent/work` (default layout) | `matches neither` |
| `spiffe://ocm-hub.example.org/ocm/cluster/cluster1/addon/my-addon/agent/my-addon-agent` | add-on user, as with the defaults |

And for `NewIDScheme` and `CheckServerID`:

| Case | Expected |
| --- | --- |
| Overlapping templates (`.../ocm/{cluster}/x/{role}` with `.../ocm/{cluster}/{addon}/{agent}`) | `can match the same SPIFFE ID` |
| No trust domains | `no agent trust domain` |
| Default server ID with default templates | ok |
| Server ID `spiffe://ocm-hub.example.org/ocm/cluster/hub/agent/work` | `agents could present it` |

Also assert the full group list for one Klusterlet case and one add-on case, so a wrong group
string fails the test.

**Run:**

```shell
dgo 'go test ./pkg/spiffe/idtemplate/ ./pkg/server/grpc/authn/ -run "Template|IDScheme|Overlap" -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

**Output:**

```text
--- PASS: TestCompile (0.00s)
--- PASS: TestRenderMatch (0.00s)
--- PASS: TestOverlaps (0.00s)
ok  	open-cluster-management.io/sdk-go/pkg/spiffe/idtemplate	0.010s
--- PASS: TestIDSchemeParse (0.00s)
--- PASS: TestIDSchemeParseNamespaceTemplate (0.00s)
--- PASS: TestNewIDScheme (0.00s)
ok  	open-cluster-management.io/sdk-go/pkg/server/grpc/authn	0.012s
```

## Task 5: commit

```shell
git add -A && git commit -s -m ":sparkles: map SPIFFE IDs to OCM identities with ID templates

Add pkg/spiffe/idtemplate to compile, render and match agent ID templates,
and IDScheme to map an agent SPIFFE ID to the OCM user and groups from
enhancement #199, plus system:authenticated. The default templates are the
proposal's scheme. Reject trust domains outside agentTrustDomains and any ID
that matches neither template."
```

## End of day

| Output | Check |
| --- | --- |
| `idtemplate.go`, `idtemplate_test.go` | tests pass |
| `spiffeid.go`, `spiffeid_test.go` | tests pass with the default and a custom template |
| `go-spiffe/v2` in `go.mod` | `go build ./...` passes |

## Rules check

- [ ] With the default templates, the mapping matches the proposal's table exactly, including
      `system:authenticated`.
- [ ] The hub's own ID can't be used as a client identity, with any accepted templates.
- [ ] `{namespace}` never changes the user or groups.

## Result (2026-11-13)

sdk-go `3c59da31`, pushed to `fork/spire`. `pkg/spiffe/idtemplate` and
`pkg/server/grpc/authn/spiffeid.go` as above, with the tests from Task 4 plus a few more rows
(dot segments, a value with a slash, `{trustDomain}` in the path).

- go-spiffe is **v2.9.0**, not v2.6.0: v2.6.0's `X509Source.GetX509BundleForTrustDomain` reads the
  bundles without the source's lock, which `-race` caught on Day 7. v2.9.0 needs grpc 1.79.3
  (sdk-go was on 1.76.0); ocm and clusteradm are already on grpc 1.84.0.
- `api` is replaced with the fork commit:
  `replace open-cluster-management.io/api => github.com/ssijbabu/api v1.3.1-0.20261113090800-b805cf6c07e4`.
- Each of the six sdk-go commits (Days 4–9) builds and passes `go vet` on its own.

## In simple words: what changed and why

Day 4 was the first code in `sdk-go`, the library that the hub and the agents share. We added
go-spiffe, the official Go library for SPIFFE, so we don't write our own code for reading and
checking SVIDs. We took version 2.9.0 and not the planned 2.6.0, because 2.6.0 has a bug where two
parts of the program read the same data at the same time; our tests found it on Day 7.

Then we wrote a small package, `idtemplate`, that understands the ID templates from Days 2 and 3.
It can do three things: check that a template is valid, fill a template in to get an ID, and take
an ID apart to get the cluster name, the role, the add-on and the agent name back. It knows
nothing about OCM, so the hub, the Klusterlet operator, the add-on framework and clusteradm can
all use the same code. If each had its own copy, they could disagree about what a template means,
and the hub would refuse agents for no visible reason.

A template has to follow some rules, and each rule is there for safety. Each placeholder fills a
whole part of the path and appears only once, so there is only one way to read an ID. At least one
part of the path is fixed text, so a template can't match every ID in a trust domain. No ID may
match both the agent template and the add-on template, because then the hub couldn't tell an
agent from an add-on. And the hub's own ID must match neither, so an agent can never show up with
the hub's identity.

Last, we wrote `IDScheme`, which turns an agent's SPIFFE ID into an OCM user and groups. These
are the same user and group names OCM already uses with the other drivers, so the permissions
that exist on the hub today work without any change. It refuses every ID from a trust domain that
is not on the list, and every ID that matches neither template. The `{namespace}` part of an ID is
only kept for logs and never used to decide what an agent may do, because the hub has no way to
check it.
