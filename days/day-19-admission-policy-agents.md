# Day 19: admission policy for the agent namespaces

**Goal:** the Klusterlet operator installs a ValidatingAdmissionPolicy and binding that make sure
only operator-owned Deployments can start pods with an agent's ServiceAccount and role label. The
operator refuses to start agents when the policy API isn't available.

**Repo:** `ocm`, branch `spire`, `manifests/klusterlet` and `pkg/operator/operators/klusterlet`.

## Before you start

| Input | Where |
| --- | --- |
| Rules | proposal, "Who may start pods in the agent namespaces", "How admission checks the chain", "Requirements" |
| Operator identity | `system:serviceaccount:open-cluster-management:klusterlet` (`deploy/klusterlet/config/operator/operator.yaml`) |
| Built-in controller identities | `system:serviceaccount:kube-system:deployment-controller`, `...:replicaset-controller` (needs `--use-service-account-credentials=true` on kube-controller-manager) |
| Pairs table | Day 18 |

Check first: the envtest kube-apiserver version in `ocm` must be 1.30 or later (see
`ENSURE_ENVTEST_SCRIPT` in `test/integration-test.mk`). If it's older, the tests below can't run
there; raise it before anything else.

## Task 1: write the policy

**New file:** `manifests/klusterlet/managed/spiffe/agent-admission-policy.yaml`. The operator
renders one per Klusterlet. `{{ .KlusterletName }}`, `{{ .AgentNamespace }}` and the pairs come
from `KlusterletConfig`.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: ocm-spiffe-agents-{{ .KlusterletName }}
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["apps"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    - apiGroups: ["batch"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["jobs", "cronjobs"]
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  variables:
  - name: user
    expression: "request.userInfo.username"
  - name: kind
    expression: "request.kind.kind"
  - name: operator
    expression: "'system:serviceaccount:open-cluster-management:klusterlet'"
  - name: deploymentController
    expression: "'system:serviceaccount:kube-system:deployment-controller'"
  - name: replicaSetController
    expression: "'system:serviceaccount:kube-system:replicaset-controller'"
  - name: pairs   # role label -> ServiceAccount, rendered per install mode
    expression: "{{ .SPIFFERolePairsCEL }}"   # e.g. {'registration': 'klusterlet-registration-sa', 'work': 'klusterlet-work-sa'}
  - name: role
    expression: >-
      has(object.metadata.labels) && 'open-cluster-management.io/spiffe-role' in object.metadata.labels
      ? object.metadata.labels['open-cluster-management.io/spiffe-role'] : ''
  - name: ownedBy
    expression: >-
      has(object.metadata.ownerReferences)
      ? object.metadata.ownerReferences.filter(o, has(o.controller) && o.controller).map(o, o.kind) : []
  validations:
  - expression: "!(variables.kind in ['StatefulSet', 'DaemonSet', 'Job', 'CronJob'])"
    message: "StatefulSets, DaemonSets, Jobs and CronJobs are not allowed in OCM agent namespaces"
  - expression: "variables.kind != 'Deployment' || variables.user == variables.operator"
    message: "only the Klusterlet operator may create or change Deployments here"
  - expression: >-
      variables.kind != 'ReplicaSet' || request.operation != 'CREATE' ||
      (variables.user == variables.deploymentController && variables.ownedBy == ['Deployment'])
    message: "ReplicaSets must be created by the Deployment controller for a Deployment"
  - expression: >-
      variables.kind != 'ReplicaSet' || request.operation != 'UPDATE' ||
      (object.spec.template == oldObject.spec.template &&
       (object.spec == oldObject.spec || variables.user == variables.deploymentController))
    message: "a ReplicaSet's Pod template can't change, and only the Deployment controller may change its spec"
  - expression: >-
      variables.kind != 'Pod' || request.operation != 'CREATE' ||
      (variables.user == variables.replicaSetController && variables.ownedBy == ['ReplicaSet'])
    message: "Pods must be created by the ReplicaSet controller for a ReplicaSet"
  - expression: >-
      variables.kind != 'Pod' || request.operation != 'CREATE' ||
      (variables.role in variables.pairs && variables.pairs[variables.role] == object.spec.serviceAccountName)
    messageExpression: "'role ' + variables.role + ' is not allowed with ServiceAccount ' + object.spec.serviceAccountName"
  - expression: >-
      variables.kind != 'Pod' || request.operation != 'UPDATE' ||
      (variables.role == (has(oldObject.metadata.labels) && 'open-cluster-management.io/spiffe-role' in oldObject.metadata.labels
                          ? oldObject.metadata.labels['open-cluster-management.io/spiffe-role'] : ''))
    message: "the spiffe-role label can't change after creation"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: ocm-spiffe-agents-{{ .KlusterletName }}
spec:
  policyName: ocm-spiffe-agents-{{ .KlusterletName }}
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: {{ .AgentNamespace }}
```

Rendered `pairs` per mode (Day 18 table):

| Mode | `SPIFFERolePairsCEL` |
| --- | --- |
| `Default`, `Hosted` | `{'registration': '<k>-registration-sa', 'work': '<k>-work-sa'}` |
| `Singleton`, `SingletonHosted` | `{'klusterlet': '<k>-work-sa'}` |

Notes:

- The ServiceAccount of a Pod can't change after creation anyway (Kubernetes refuses it), so only
  the label needs the update rule.
- Status updates go through the `status` subresource; the ReplicaSet rule allows them because the
  spec doesn't change. Check that the Pod rules don't block kubelet status updates (they don't
  touch the role label).
- The operator itself creates Secrets, ServiceAccounts and so on in the namespace; those resources
  aren't matched.

## Task 2: the operator installs it and checks it

In the Klusterlet controller, in `spiffe` mode, before applying agent Deployments:

```go
// ensureAdmissionPolicy installs the policy and binding and reports whether agents may run.
func (n *klusterletController) ensureAdmissionPolicy(ctx context.Context, k *operatorapiv1.Klusterlet, cfg KlusterletConfig) error {
	served, err := apiServed(n.discovery, "admissionregistration.k8s.io/v1", "validatingadmissionpolicies")
	if err != nil {
		return err
	}
	if !served {
		return degraded("SPIFFEAdmissionPolicyUnavailable",
			"spiffe mode needs the admissionregistration.k8s.io/v1 ValidatingAdmissionPolicy API (Kubernetes 1.30 or later)")
	}
	return n.applyAdmissionObjects(ctx, k, cfg) // policy + binding, from the manifest above
}
```

If it returns an error, the operator applies no agent Deployment and sets `Degraded=True` with
the reason and message. Also add the Klusterlet operator RBAC for
`validatingadmissionpolicies` and `validatingadmissionpolicybindings` (create, get, update,
delete, watch) to `deploy/klusterlet/config/rbac`.

## Task 3: tests in envtest

There's no kube-controller-manager in envtest, so the test plays the controllers by impersonating
their ServiceAccounts:

```go
func as(user string) *kubernetes.Clientset {
	cfg := rest.CopyConfig(envtestCfg)
	cfg.Impersonate = rest.ImpersonationConfig{UserName: user}
	return kubernetes.NewForConfigOrDie(cfg)
}
```

**New file:** `test/integration/operator/spiffe_admission_test.go`. Apply the rendered policy for
`Default` mode, Klusterlet `klusterlet`, namespace `open-cluster-management-agent`.

| # | Request (as user) | Expected |
| --- | --- | --- |
| 1 | create Deployment `x` (operator) | allowed |
| 2 | create Deployment `y` (`alice`) | denied: `only the Klusterlet operator may create or change Deployments here` |
| 3 | create ReplicaSet owned by Deployment `x` (deployment-controller) | allowed |
| 4 | create ReplicaSet with no owner (deployment-controller) | denied |
| 5 | update that ReplicaSet's image (deployment-controller) | denied: `Pod template can't change` |
| 6 | scale that ReplicaSet to 2 (deployment-controller) | allowed |
| 7 | scale it (`alice`) | denied |
| 8 | create Pod, owner ReplicaSet, SA `klusterlet-work-sa`, role `work` (replicaset-controller) | allowed |
| 9 | same Pod, role `klusterlet` | denied: `role klusterlet is not allowed with ServiceAccount klusterlet-work-sa` |
| 10 | same Pod (`alice`) | denied |
| 11 | change role label on Pod from #8 (operator) | denied |
| 12 | create Job, DaemonSet, StatefulSet, CronJob (operator) | all denied |
| 13 | Singleton pairs: Pod role `work` with `klusterlet-work-sa` | denied |
| 14 | Hosted, namespace `cluster4`: Pod role `registration` with `cluster4-registration-sa` | allowed |
| 15 | Pod in another namespace, any role (`alice`) | allowed (not matched) |

Policies take a moment to become active after creation. Use `Eventually` on a request that must be
denied before running the table.

Operator test (unit, fake discovery):

| Input | Expected |
| --- | --- |
| discovery without `validatingadmissionpolicies` | no agent Deployment applied; `Degraded=True`, reason `SPIFFEAdmissionPolicyUnavailable` |
| API served | policy and binding applied; agents applied |
| policy deleted | recreated on the next reconcile (recovery comes on Day 21) |

**Run:**

```shell
dgo 'make test-registration-operator-integration ARGS="-ginkgo.focus=admission"' 2>&1 | tail -3
dgo 'go test ./pkg/operator/operators/klusterlet/... -run Admission -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
```

**Output:**

```text
Ran 15 of 74 Specs in 31.0 seconds
SUCCESS! -- 15 Passed | 0 Failed | 0 Pending | 59 Skipped
```

## Task 4: commit

```shell
git add -A && git commit -s -m ":sparkles: admission policy for SPIFFE agent namespaces"
```

## Rules check

- [ ] Only operator-owned Deployments can produce pods with agent ServiceAccounts and role labels.
- [ ] Without the policy API, no agent starts.

## Status (2026-11-13)

Done, committed locally. See "Progress (Days 10-23)" in `questions.md` for the commit and findings.

## In simple words: what changed and why

Day 19 dealt with a risk that comes with SPIRE. SPIRE gives an agent's SVID to any pod that has
the right namespace, ServiceAccount and label. So anyone who can start such a pod gets the
agent's identity and can talk to the hub as that cluster. With the other drivers the credential
is in a Secret, and reading the Secret is what has to be protected. Here, starting pods is what
has to be protected.

The operator now installs an admission policy for the agent namespace before any agent runs. An
admission policy is a rule that the Kubernetes API server checks on every request. This one says:
only the operator may create or change Deployments there; pods may only come from those
Deployments, through Kubernetes' own controllers; each pod's role label must match its
ServiceAccount for the install mode; and other kinds of workload (StatefulSet, DaemonSet, Job,
CronJob) are not allowed at all. So the only pods that can get an agent's SVID are the ones the
operator created.

If the cluster doesn't support this kind of policy, the operator deploys no agents and reports
why. Running agents without the policy would mean their identities are not protected.

The rules are tested in a real API server. The test acts as the operator, as the built-in
controllers and as an ordinary user, and checks who is allowed and who is refused. Days 19, 20
and 21 are one commit (`c77e17fd1`).
