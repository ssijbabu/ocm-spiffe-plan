# Day 20: images, exec, add-on namespaces and namespace labels

**Goal:** finish the admission rules: no image changes or ephemeral containers on running agent
pods, no `exec`/`attach`, the add-on namespace rules, and the rules for the
`open-cluster-management.io/spiffe-addon-namespace` label.

**Repo:** `ocm`, branch `spire`.

## Before you start

| Input | Where |
| --- | --- |
| Agent policy | Day 19 |
| Add-on rules | proposal, Add-on namespaces, "Add-on namespaces must be dedicated", "Which namespaces recovery cleans" |
| Work agent identity on the managed cluster | `Default`/`Singleton`: `system:serviceaccount:open-cluster-management-agent:<k>-work-sa`. Hosted: the user in the derived managed-cluster kubeconfig; find which ServiceAccount the operator creates on the managed cluster for it |

## Task 1: more rules for the agent policy

**Add to** the Day 19 policy's `matchConstraints.resourceRules`:

```yaml
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["UPDATE"]
      resources: ["pods/ephemeralcontainers"]
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CONNECT"]
      resources: ["pods/exec", "pods/attach"]
```

**Add validations:**

```yaml
  - expression: "!(request.subResource in ['exec', 'attach'])"
    message: "exec and attach are not allowed on OCM agent pods; read the logs instead"
  - expression: "request.subResource != 'ephemeralcontainers'"
    message: "ephemeral containers are not allowed on OCM agent pods"
  - expression: >-
      variables.kind != 'Pod' || request.operation != 'UPDATE' || request.subResource != '' ||
      (object.spec.containers.map(c, c.image) == oldObject.spec.containers.map(c, c.image) &&
       (has(object.spec.initContainers) ? object.spec.initContainers.map(c, c.image) : []) ==
       (has(oldObject.spec.initContainers) ? oldObject.spec.initContainers.map(c, c.image) : []))
    message: "container images of an agent pod can't change"
```

For `CONNECT`, the object is a `PodExecOptions`/`PodAttachOptions`, and `request.kind.kind` is
that kind, not `Pod`. The rules above only use `request.subResource`, so they work. `pods/log` is
a `GET`, which admission never sees, so logs stay readable.

## Task 2: the add-on namespace policy

**New file:** `manifests/klusterlet/managed/spiffe/addon-admission-policy.yaml`, one per Klusterlet.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: ocm-spiffe-addons-{{ .KlusterletName }}
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:   # same kinds as the agent policy, plus exec/attach/ephemeralcontainers
    - { apiGroups: ["apps"], apiVersions: ["v1"], operations: ["CREATE","UPDATE"], resources: ["deployments","replicasets","statefulsets","daemonsets"] }
    - { apiGroups: ["batch"], apiVersions: ["v1"], operations: ["CREATE","UPDATE"], resources: ["jobs","cronjobs"] }
    - { apiGroups: [""], apiVersions: ["v1"], operations: ["CREATE","UPDATE"], resources: ["pods"] }
    - { apiGroups: [""], apiVersions: ["v1"], operations: ["UPDATE"], resources: ["pods/ephemeralcontainers"] }
    - { apiGroups: [""], apiVersions: ["v1"], operations: ["CONNECT"], resources: ["pods/exec","pods/attach"] }
  variables:
  - name: user
    expression: "request.userInfo.username"
  - name: kind
    expression: "request.kind.kind"
  - name: workAgent
    expression: "'{{ .SPIFFEWorkAgentUser }}'"   # e.g. system:serviceaccount:open-cluster-management-agent:klusterlet-work-sa
  - name: klusterletSAs
    expression: "{{ .SPIFFEKlusterletSAsCEL }}"  # e.g. ['klusterlet-registration-sa', 'klusterlet-work-sa']
  - name: controllerFor   # who may create a Pod, by owner kind
    expression: >-
      {'ReplicaSet': 'system:serviceaccount:kube-system:replicaset-controller',
       'DaemonSet': 'system:serviceaccount:kube-system:daemon-set-controller',
       'StatefulSet': 'system:serviceaccount:kube-system:statefulset-controller'}
  - name: ownedBy
    expression: >-
      has(object.metadata.ownerReferences)
      ? object.metadata.ownerReferences.filter(o, has(o.controller) && o.controller).map(o, o.kind) : []
  - name: role
    expression: >-
      has(object.metadata.labels) && 'open-cluster-management.io/spiffe-role' in object.metadata.labels
      ? object.metadata.labels['open-cluster-management.io/spiffe-role'] : ''
  validations:
  - expression: "!(variables.kind in ['Job', 'CronJob'])"
    message: "Jobs and CronJobs are not allowed in add-on namespaces"
  - expression: "!(variables.kind in ['Deployment', 'DaemonSet', 'StatefulSet']) || variables.user == variables.workAgent"
    message: "only this Klusterlet's work agent may create or change add-on workloads"
  - expression: >-
      variables.kind != 'ReplicaSet' || request.operation != 'CREATE' ||
      (variables.user == 'system:serviceaccount:kube-system:deployment-controller' && variables.ownedBy == ['Deployment'])
    message: "ReplicaSets must be created by the Deployment controller for a Deployment"
  - expression: >-
      variables.kind != 'Pod' || request.operation != 'CREATE' ||
      (size(variables.ownedBy) == 1 && variables.ownedBy[0] in variables.controllerFor &&
       variables.user == variables.controllerFor[variables.ownedBy[0]])
    message: "Pods must be created by the matching built-in controller"
  - expression: >-
      variables.kind != 'Pod' || request.operation != 'CREATE' ||
      (variables.role == 'addon' && !(object.spec.serviceAccountName in variables.klusterletSAs))
    message: "add-on pods must have spiffe-role addon and their own ServiceAccount"
  # + the ReplicaSet template, image, exec/attach and ephemeral rules from the agent policy
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: ocm-spiffe-addons-{{ .KlusterletName }}
spec:
  policyName: ocm-spiffe-addons-{{ .KlusterletName }}
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        open-cluster-management.io/spiffe-addon-namespace: {{ .KlusterletName }}
```

The label value names the Klusterlet, so in hosted mode `cluster4`'s policy only covers
`cluster4`'s add-on namespaces, and only `cluster4`'s work agent may create workloads there.

## Task 3: the namespace label policy

One cluster-wide policy for all Klusterlets the operator manages.

**New file:** `manifests/klusterlet/managed/spiffe/namespace-label-policy.yaml`

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: ocm-spiffe-addon-namespace-label
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - { apiGroups: [""], apiVersions: ["v1"], operations: ["CREATE", "UPDATE"], resources: ["namespaces"] }
  variables:
  - name: key
    expression: "'open-cluster-management.io/spiffe-addon-namespace'"
  - name: newValue
    expression: "has(object.metadata.labels) && variables.key in object.metadata.labels ? object.metadata.labels[variables.key] : ''"
  - name: oldValue
    expression: >-
      request.operation == 'UPDATE' && has(oldObject.metadata.labels) && variables.key in oldObject.metadata.labels
      ? oldObject.metadata.labels[variables.key] : ''
  - name: operator
    expression: "'system:serviceaccount:open-cluster-management:klusterlet'"
  - name: workAgents   # Klusterlet name -> work agent user, rendered from all Klusterlets
    expression: "{{ .SPIFFEWorkAgentsCEL }}"   # e.g. {'klusterlet': 'system:serviceaccount:open-cluster-management-agent:klusterlet-work-sa'}
  validations:
  - expression: >-
      request.operation != 'CREATE' || variables.newValue == '' ||
      request.userInfo.username == variables.operator ||
      (variables.newValue in variables.workAgents && request.userInfo.username == variables.workAgents[variables.newValue])
    message: "only the Klusterlet operator or that Klusterlet's work agent may create an add-on namespace"
  - expression: >-
      request.operation != 'UPDATE' || variables.newValue == variables.oldValue ||
      (variables.newValue == '' && request.userInfo.username == variables.operator)
    message: "the spiffe-addon-namespace label can't be added or changed on an existing namespace; only the operator may remove it"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: ocm-spiffe-addon-namespace-label
spec:
  policyName: ocm-spiffe-addon-namespace-label
  validationActions: ["Deny"]
```

No `matchResources` on the binding: it covers every Namespace request, so the label can't be put
on `default` or any other namespace.

## Task 4: tests (envtest, impersonation as on Day 19)

| # | Request (as user) | Expected |
| --- | --- | --- |
| 1 | `CONNECT pods/exec` on an agent pod (`alice`, bound to `admin` in the namespace) | denied: `exec and attach are not allowed` |
| 2 | `CONNECT pods/attach` (cluster-admin) | denied |
| 3 | `GET pods/log` (`alice`) | allowed |
| 4 | change an agent pod's image (operator) | denied |
| 5 | add an ephemeral container (cluster-admin) | denied |
| 6 | add-on ns `addon-a` labeled `klusterlet`: Deployment by work agent | allowed |
| 7 | same, DaemonSet by `alice` | denied |
| 8 | Pod owned by DaemonSet, created by daemon-set-controller, role `addon`, SA `my-addon-agent-sa` | allowed |
| 9 | same Pod with SA `klusterlet-work-sa` | denied |
| 10 | Job in `addon-a` (work agent) | denied |
| 11 | create namespace `addon-b` with label `klusterlet` (work agent) | allowed |
| 12 | create namespace `addon-c` with label `klusterlet` (`alice`) | denied |
| 13 | add label `klusterlet` to `default` (cluster-admin) | denied |
| 14 | change label on `addon-b` to `cluster4` (work agent) | denied |
| 15 | remove label from `addon-b` (operator) | allowed |
| 16 | remove label (`alice`) | denied |
| 17 | hosted: two Klusterlets `cluster4`, `cluster5`; `cluster5`'s work agent creates a Deployment in `cluster4`'s add-on namespace | denied |
| 18 | Deployment in `default` (`alice`) | allowed (no label, not covered) |

Example for #1 (exec through the API):

```go
req := as("alice").CoreV1().RESTClient().Post().Resource("pods").Namespace(ns).Name(pod).
	SubResource("exec").Param("command", "true").Param("stdout", "true")
_, err := remotecommand.NewSPDYExecutor(aliceCfg, "POST", req.URL())   // create executor
// ...Stream(...) returns an error containing "exec and attach are not allowed"
```

If streaming is awkward in envtest, use a raw `POST` to the exec URL and check for `403` with the
policy message.

**Run:**

```shell
dgo 'make test-registration-operator-integration ARGS="-ginkgo.focus=admission"' 2>&1 | tail -3
```

**Output:** the 15 specs from Day 19 plus these 18 pass.

## Task 5: commit

```shell
git add -A && git commit -s -m ":sparkles: block exec, image changes and foreign workloads in SPIFFE agent and add-on namespaces"
```

## Rules check

- [ ] No user, including cluster-admin, can run a process in an agent pod through the API.
- [ ] The add-on label can't be added to an existing namespace.

## Status (2026-11-13)

Done, committed locally. See "Progress (Days 10-23)" in `questions.md` for the commit and findings.

## In simple words: what changed and why

Day 20 finished the admission rules. First, more rules for agent pods: nobody can run a command
in one (`exec`), attach to one, add a debug container to one, or change its image or its role
label. Each of these would put someone's own code inside a pod that has an agent's SVID, and that
code could read the SVID from the socket. The rule applies to everyone, including a cluster
admin. Reading logs still works.

Second, a policy for add-on namespaces. Add-on agents get SVIDs too, so their namespaces need the
same care. A namespace counts as an add-on namespace when it has a label that names the
Klusterlet. In such a namespace only that Klusterlet's work agent may create workloads, because
that is how add-ons are installed. Add-on pods must have the role `addon` and their own
ServiceAccount, so an add-on can't get an agent's identity.

Third, a policy for the label itself. The label is what tells SPIRE and OCM "pods here may get
add-on identities", so it must not be easy to add. It can only be set when the namespace is
created, and only by the operator or that Klusterlet's work agent. Only the operator can remove
it. This stops someone from putting the label on an existing namespace such as `default`, which
would let the pods already there ask for add-on identities.

In the hosted modes, where one management cluster carries several Klusterlets, the label value
keeps them apart: one Klusterlet's work agent can't create anything in another Klusterlet's add-on
namespace.
