# Day 21: recovery and trusted state

**Goal:** the operator knows when it can trust the agent and add-on namespaces, runs the recovery
steps when it can't, and sets `SPIFFEAdmissionReady` when it's done.

**Repo:** `ocm`, branch `spire`, `pkg/operator/operators/klusterlet`.

## Before you start

| Input | Where |
| --- | --- |
| Status fields | Day 3: `status.spiffe.admissionPolicy`, `status.spiffe.addOnNamespace`, `SPIFFEAdmissionReady` |
| Policies | Days 19 and 20 |
| Design | proposal, "When the operator trusts the namespaces", "Which namespaces recovery cleans", recovery steps 1 to 4 |

## Task 1: decide when to run recovery

```go
type observed struct {
	PolicyUID, BindingUID               types.UID
	PolicyGeneration, BindingGeneration int64
}

// needsRecovery compares the live policy/binding with what the Klusterlet status records.
func needsRecovery(rec *operatorapiv1.SPIFFEAdmissionPolicyStatus, live observed) bool {
	return rec == nil ||
		rec.PolicyUID != live.PolicyUID || rec.PolicyGeneration != live.PolicyGeneration ||
		rec.BindingUID != live.BindingUID || rec.BindingGeneration != live.BindingGeneration
}
```

Call it on every reconcile, also on the first one after the operator starts. Don't keep anything in
memory between reconciles: the status is the only record.

If the operator changes the policy itself (operator upgrade, different rendered spec), apply it and
record the new generation *without* recovery, but only when the values before the apply matched
the record:

```go
before := liveObserved()
if !needsRecovery(rec, before) && specDiffers(rendered, live) {
	apply(rendered)
	return recordStatus(liveObserved())   // generation moved because of us
}
```

## Task 2: the four steps

```go
func (r *recoverer) run(ctx context.Context, k *operatorapiv1.Klusterlet) error {
	setCondition(k, SPIFFEAdmissionReady, false, "Recovering", "cleaning agent and add-on namespaces")

	// 1. Stop the operator's own agents, work agent included, and wait for their pods to go.
	if err := r.scaleAgents(ctx, k, 0); err != nil { return err }
	if err := r.waitNoPods(ctx, r.agentNamespace(k), agentSelector); err != nil { return err }

	// 2. Restore the policies and note what we restored.
	if err := r.applyPolicies(ctx, k); err != nil { return err }
	noted := r.liveObserved(ctx, k)
	trusted, err := r.trustedAddOnNamespaces(ctx, k)
	if err != nil { return err }
	if err := r.unlabelUntrusted(ctx, k, trusted); err != nil { return err }

	// 3. Delete every workload in the agent namespace and the trusted add-on namespaces.
	for _, ns := range append([]string{r.agentNamespace(k)}, trusted...) {
		if err := r.deleteWorkloads(ctx, ns); err != nil { return err }
		if err := r.waitNoPods(ctx, ns, labels.Everything()); err != nil { return err }
	}

	// 4. Start over if the policies changed during cleanup; otherwise record and restart agents.
	if r.liveObserved(ctx, k) != noted {
		return errRestartRecovery
	}
	if err := r.recordStatus(ctx, k, noted); err != nil { return err }
	setCondition(k, SPIFFEAdmissionReady, true, "Recorded", "admission state recorded")
	return r.scaleAgents(ctx, k, 1)
}
```

`deleteWorkloads` deletes Deployments, ReplicaSets, DaemonSets, StatefulSets and Pods with
`PropagationPolicy: Foreground`. In the agent namespace it then re-creates its own agent
Deployments in step 4 (from the normal reconcile).

Waiting: don't block the reconcile for minutes. Return a "requeue after 5s" result and continue
from where you were next time; record progress in a condition (`Recovering`, step N).

## Task 3: which add-on namespaces are trusted

```go
// trustedAddOnNamespaces returns the namespaces carrying this Klusterlet's label whose UID is on
// record: created by this Klusterlet's work agent (listed in one of its AppliedManifestWorks), or
// the open-cluster-management-agent-addon namespace the operator created.
func (r *recoverer) trustedAddOnNamespaces(ctx context.Context, k *operatorapiv1.Klusterlet) ([]string, error) {
	recorded := map[types.UID]bool{}
	if k.Status.SPIFFE != nil && k.Status.SPIFFE.AddOnNamespace != nil {
		recorded[k.Status.SPIFFE.AddOnNamespace.UID] = true
	}
	amws, err := r.workClient.WorkV1().AppliedManifestWorks().List(ctx, metav1.ListOptions{})
	if err != nil {
		return nil, err
	}
	for _, amw := range amws.Items {
		if !strings.HasPrefix(amw.Name, r.hubHash(k)+"-") { // this Klusterlet's AMWs; check the real naming
			continue
		}
		for _, res := range amw.Status.AppliedResources {
			if res.Group == "" && res.Resource == "namespaces" {
				recorded[types.UID(res.UID)] = true
			}
		}
	}
	nss, err := r.kubeClient.CoreV1().Namespaces().List(ctx, metav1.ListOptions{
		LabelSelector: "open-cluster-management.io/spiffe-addon-namespace=" + k.Name,
	})
	if err != nil {
		return nil, err
	}
	var out []string
	for _, ns := range nss.Items {
		if recorded[ns.UID] {
			out = append(out, ns.Name)
		}
	}
	return out, nil
}
```

`unlabelUntrusted` removes this Klusterlet's label from every labeled namespace that isn't in
`trusted`. The namespace-label policy (Day 20) allows the operator to remove it.

The operator creates `open-cluster-management-agent-addon` itself only when it doesn't exist, with
the label, and records its name and UID in `status.spiffe.addOnNamespace`. It never labels an
existing namespace. If the namespace exists without the label, set a condition
(`AddOnNamespaceNotDedicated`) and leave it alone.

## Task 4: tests

**Unit** (fake clients, fake clock):

| Case | Input | Expected |
| --- | --- | --- |
| Nothing recorded | first activation | recovery runs; at the end status has the 4 values, `SPIFFEAdmissionReady=True` |
| Recorded, unchanged | same UIDs and generations | no recovery |
| Policy recreated | new UID | recovery |
| Policy edited | generation +1 | recovery |
| Operator's own change | rendered spec differs, record matches before apply | apply, record new generation, no recovery |
| Changed during cleanup | generation changes between step 2 and 4 | `errRestartRecovery`; status not recorded |
| Untrusted label on `default` | `default` labeled `klusterlet`, UID not on record | nothing in `default` deleted; label removed |
| Other Klusterlet's namespace labeled with ours | UID not in our AMWs | not cleaned; label removed |
| Existing unlabeled `open-cluster-management-agent-addon` | namespace exists | not labeled; condition `AddOnNamespaceNotDedicated` |

**Integration** (envtest; Pods won't actually run, so "wait for pods to go" sees deletions only):

| Scenario | Steps | Expected |
| --- | --- | --- |
| First activation | create Deployment `evil` with `klusterlet-work-sa` and role `work` in the agent namespace, then switch the Klusterlet to `spiffe` | `evil` deleted before agent Deployments exist; `SPIFFEAdmissionReady=True` |
| Operator down | stop the operator; delete and recreate the policy; start it | recovery runs (new UID) |
| DaemonSet add-on | add-on DaemonSet in a trusted add-on ns; delete the policy | DaemonSet deleted, agents at 0 until done; after recovery the work agent re-applies it |
| Upgrade | change the rendered policy (simulate a new operator version) | no recovery |

**Run:**

```shell
dgo 'go test ./pkg/operator/operators/klusterlet/... -run "Recovery|Trusted" -race -v' 2>&1 | grep -E '^(--- |ok|FAIL)'
dgo 'make test-registration-operator-integration ARGS="-ginkgo.focus=recovery"' 2>&1 | tail -3
```

## Task 5: commit

```shell
git add -A && git commit -s -m ":sparkles: recover and record trusted admission state for SPIFFE agents"
```

## Rules check

- [ ] `SPIFFEAdmissionReady` is only `True` after a finished recovery.
- [ ] Recovery never deletes anything outside the agent namespace and recorded add-on namespaces.

## Status (2026-11-13)

Done, committed locally. See "Progress (Days 10-23)" in `questions.md` for the commit and findings.

## In simple words: what changed and why

Day 21 answered the question "what if someone removes or changes the policies?". The operator
writes into the Klusterlet status which policies it installed, with their UID and generation (a
number that goes up on every change). This is the record from Day 3. It is the only record:
nothing is kept in memory, so it still works after the operator restarts.

On every pass the operator compares the live policies with the record. If one was deleted,
replaced or edited by someone else, the operator can no longer trust the namespaces. While the
policy was missing, someone could have started a pod that poses as an agent, and the operator
can't tell a good pod from a bad one. So it runs recovery: it restores the policies, removes the
add-on label from namespaces that are not on record, deletes every workload in the agent
namespace and the recorded add-on namespaces, waits until no pod is left, writes the new record,
and only then starts its own agents again. The `SPIFFEAdmissionReady` condition is True only
after this has finished, which is the signal that the SPIRE entries are safe to create.

Recovery never deletes anything outside those namespaces. When the operator changes a policy
itself, for example after an upgrade, it records the new state without recovery, because nobody
else was involved. And it creates the add-on namespace with the label only when the namespace
doesn't exist yet; it never labels a namespace that is already there.

The e2e tests found three bugs here, all fixed. A failed status write lost the record, so the
operator started the agents and deleted them again in a loop; agents now start only after the
record is stored. Deleting a Klusterlet left its policies behind; they are now removed. And a
deleted policy went unnoticed for up to five minutes, a window in which a fake agent pod could
get an SVID; the operator now watches the policies and reacts within seconds. One limit remains:
in the hosted modes, the policies on the managed cluster are still only checked every five minutes
(`questions.md` Q20).
