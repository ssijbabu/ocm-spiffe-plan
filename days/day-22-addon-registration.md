# Day 22: add-on registration, spoke and hub

**Goal:** with the `spiffe` add-on driver, the registration agent creates no add-on credentials and
only reports `driver: spiffe`; it refuses `customSigner` registrations and non-dedicated install
namespaces. On the hub, the add-on manager checks that an add-on's `kubeClient` subject is one an
SVID can carry. Then the ocm draft PR is opened.

**Repo:** `ocm`, branch `spire`, `pkg/registration/spoke/addon`, `pkg/registration/register`,
`pkg/addon`.

## Before you start

| Input | Where |
| --- | --- |
| Spoke add-on controller | `pkg/registration/spoke/addon/registration_controller.go`: `ensureDriver` sets `status.registrations[].kubeClient.driver`; `addonDriverFactory.Fork(addOnName, authConfig, secretOption)` creates the credential driver |
| Add-on driver option | `factory.Options.AddonKubeClientRegistrationAuth` (`csr` or `token`), flag `--addon-kubeclient-registration-auth` |
| Hub side | `pkg/addon/templateagent/registration.go` and the add-on manager controllers that set `RegistrationApplied` |

## Task 1: spoke, the `spiffe` add-on driver

1. Accept `spiffe` in `factory.Options.Validate()`:

   ```go
   switch s.AddonKubeClientRegistrationAuth {
   case "", "csr", "token", "spiffe":
   default:
       return fmt.Errorf("unsupported addon-kubeclient-registration-auth: %s", s.AddonKubeClientRegistrationAuth)
   }
   ```

2. `ensureDriver` writes `spiffe` to `status.registrations[].kubeClient.driver`.

3. In the place that calls `Fork` for a `KubeClient` registration: with `spiffe`, don't fork a
   credential driver at all. No CSR, no token request, no `<addon>-hub-kubeconfig` Secret. Set
   `RegistrationApplied=True` and stop.

4. For a `customSigner` registration with `spiffe`, set:

   ```yaml
   - type: RegistrationApplied
     status: "False"
     reason: CustomSignerNotSupported
     message: "customSigner registrations are not supported with the spiffe driver: the hub does not sign certificates"
   ```

5. Dedicated namespace: read the add-on's install namespace. If it lacks
   `open-cluster-management.io/spiffe-addon-namespace: <klusterlet>`, set:

   ```yaml
   - type: RegistrationApplied
     status: "False"
     reason: AddOnNamespaceNotDedicated
     message: "install namespace default is not a dedicated add-on namespace; move my-addon to a namespace of its own"
   ```

## Task 2: hub, the subject check

**Rule:** the hub maps an add-on agent's SVID to:

```text
user:   system:open-cluster-management:cluster:<cluster>:addon:<addon>:agent:<agent>
groups: system:open-cluster-management:cluster:<cluster>:addon:<addon>
        system:open-cluster-management:addon:<addon>
        system:authenticated
```

So a `kubeClient` subject works only if its user matches that pattern for this cluster and add-on
(any agent name), and every group is in that list.

```go
// subjectCarriedBySVID reports whether an SVID-mapped identity can present subject.
func subjectCarriedBySVID(cluster, addon string, s addonv1beta1.KubeClientSubject) bool {
	prefix := fmt.Sprintf("system:open-cluster-management:cluster:%s:addon:%s:agent:", cluster, addon)
	if !strings.HasPrefix(s.User, prefix) || strings.Contains(strings.TrimPrefix(s.User, prefix), ":") {
		return false
	}
	allowed := sets.New(
		fmt.Sprintf("system:open-cluster-management:cluster:%s:addon:%s", cluster, addon),
		"system:open-cluster-management:addon:"+addon,
		"system:authenticated",
	)
	return allowed.HasAll(s.Groups...)
}
```

Apply it where the add-on manager sets `RegistrationApplied` for clusters whose add-on driver is
`spiffe`:

| `kubeClient` subject | Result |
| --- | --- |
| user `system:open-cluster-management:cluster:cluster1:addon:my-addon:agent:my-addon-agent`, groups the two add-on groups | `RegistrationApplied=True` |
| same plus `system:authenticated` | `True` |
| user `my-team-bot` | `False`, reason `SubjectNotSupportedBySPIFFE` |
| right user, extra group `admins` | `False` |

## Task 3: tests

**Unit, spoke** (`registration_controller_test.go`):

| Case | Input | Expected |
| --- | --- | --- |
| kubeClient, spiffe | MCA `my-addon` with one `kubeClient` registration; ns labeled `klusterlet` | status `driver: spiffe`, `RegistrationApplied=True`; no Secret, CSR or TokenRequest action |
| customSigner, spiffe | `customSigner` registration | `RegistrationApplied=False`, `CustomSignerNotSupported` |
| shared namespace | install namespace `default`, unlabeled | `RegistrationApplied=False`, `AddOnNamespaceNotDedicated` |
| csr, unchanged | driver `csr` | existing behaviour |

Expected status for the first case:

```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ManagedClusterAddOn
metadata:
  name: my-addon
  namespace: cluster1
status:
  registrations:
  - type: kubeClient
    kubeClient:
      driver: spiffe
  conditions:
  - type: RegistrationApplied
    status: "True"
```

**Unit, hub:** the subject table from Task 2.

**Integration:** add a spec to `test/integration/registration/addon_registration_test.go` (or a new
`addon_spiffe_registration_test.go`): enable `my-addon` on a SPIFFE cluster, then an add-on agent
with SVID `.../cluster/<c>/addon/my-addon/agent/my-addon-agent` updates the add-on's status; rotate
its SVID; delete the MCA and check its connection is closed (Day 12).

**Run:**

```shell
dgo 'go test ./pkg/registration/spoke/addon/... ./pkg/addon/... -run "SPIFFE|Spiffe" -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'make test-registration-integration ARGS="-ginkgo.focus=spiffe"' 2>&1 | tail -2
```

## Task 4: commit, push, open the ocm draft PR

```shell
git add -A && git commit -s -m ":sparkles: spiffe add-on registration, no add-on credentials on the spoke"
git push origin spire
gh pr create -R open-cluster-management-io/ocm --draft --head ssijbabu:spire \
  --title ":sparkles: SPIFFE gRPC registration" \
  --body "Implements enhancement #199 in ocm (open-cluster-management-io/enhancements#203).
Depends on open-cluster-management-io/api#<n> and open-cluster-management-io/sdk-go#<n>.
The replace lines in go.mod are temporary."
```

## Rules check

- [ ] No add-on credential of any kind is created on the spoke.
- [ ] A subject an SVID can't carry is reported, not silently ignored.

## Update: multiple agent trust domains

Add-on agents must be in the trust domain their cluster is pinned to. In hosted modes that is the management cluster's: a hosted add-on on the management cluster is in it; an add-on agent on the managed cluster itself is accepted only if that cluster's SPIRE is also in that trust domain.

## Update: agent ID templates

No change. The hub's subject check compares the `kubeClient` subject with the user and groups an
add-on SVID maps to. The add-on template changes only the layout of the ID, never the user and
groups, so a default subject still passes and a custom user such as `my-team-bot` still fails.

## Status (2026-11-13)

Done, committed locally. See "Progress (Days 10-23)" in `questions.md` for the commit and findings.

## In simple words: what changed and why

Day 22 brought add-ons into the model. With the other drivers, the registration agent gets a
credential for each add-on from the hub and stores it in a Secret named `<addon>-hub-kubeconfig`.
With the `spiffe` add-on driver it creates no certificate request, no token and no Secret. Each
add-on agent has its own SVID from SPIRE. The add-on driver must be `spiffe` exactly when the
agents use `spiffe`, the same rule the API enforces since Day 3.

Two kinds of add-on setup are refused, with a clear reason in the add-on's status. Add-ons that
ask for a certificate from a custom signer can't work, because in this mode nothing signs
certificates. And an add-on whose install namespace doesn't have the dedicated add-on label is
refused, because in a shared namespace other pods could get the add-on's identity.

An add-on agent still needs to know a few things: the hub's address, the hub's SPIFFE ID, where
the SPIRE socket is, and its own SPIFFE ID. The registration agent writes these into a ConfigMap
named `<addon>-spiffe` in the add-on's namespace, and deletes it when the add-on is removed.
Nothing in it is secret: it is addresses and names, no keys and no certificates. So it stays
within the rule that OCM stores no credentials (`questions.md` Q8).

The integration test runs an add-on agent with its own SVID. It can update its own add-on's
status, it can't touch another add-on, and it is cut off when its add-on is disabled. The plan
also had a hub-side check of the add-on's subject; that was built in the add-on framework on
Day 23. The draft pull request for `ocm` is not opened yet, because nothing is pushed.
