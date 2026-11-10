# Day 1: set up the workspace

**Goal:** every repo is on a clean `spire` branch, builds, and has a saved baseline of test results.
Nothing is changed in the code today.

**Repos:** `api`, `sdk-go`, `ocm`, `clusteradm`, and `addon-framework` (cloned today).

## Before you start

| Input | Where |
| --- | --- |
| Workspace | `/home/ssijbabu/Documents/repos/ocm` |
| Design | `enhancements/enhancements/sig-architecture/199-spiffe-grpc-registration/README.md`, branch `spire` |
| Go wrapper | `plan/tools/dgo` (Go is not installed on the host; this runs `golang:1.27` in Docker) |
| Remotes | `api`, `clusteradm`, `sdk-go`: `origin` = upstream, `fork` = `ssijbabu/*`. `ocm`, `enhancements`: `origin` = fork, `upstream` = upstream |

## Task 1: put every repo on a fresh `spire`

**Input:** the four existing repos, each with a `spire` branch at some older upstream commit.

**Steps:**

```shell
cd /home/ssijbabu/Documents/repos/ocm
for r in api sdk-go clusteradm; do
  git -C $r fetch origin && git -C $r switch spire && git -C $r merge --ff-only origin/main
done
git -C ocm fetch upstream && git -C ocm switch spire && git -C ocm merge --ff-only upstream/main
for r in api sdk-go ocm clusteradm; do
  echo "$r: $(git -C $r branch --show-current) $(git -C $r log -1 --format='%h %s')"
done
```

**Output (example):**

```text
api: spire 3e02a5dd :sparkles: Add per-agent bind configuration to AgentConfiguration (#467)
sdk-go: spire 592fbe65 chore: update ocm api to v1.4.0 (#237)
ocm: spire 07771b3bf chore: bump k8s.io dependencies to v0.37.2 (#1793)
clusteradm: spire ce33b120 :seedling: Bump github.com/onsi/ginkgo/v2 (#599)
```

If `--ff-only` fails, `spire` has local commits. Look at them with
`git log origin/main..spire` before deciding anything; don't reset them away.

## Task 2: clone addon-framework

**Input:** nothing local yet.

**Steps:**

```shell
cd /home/ssijbabu/Documents/repos/ocm
git clone https://github.com/open-cluster-management-io/addon-framework.git
cd addon-framework
gh repo fork --remote --remote-name fork        # creates ssijbabu/addon-framework if missing
git switch -c spire
git config user.name ssijbabu && git config user.email ssijbabu@gmail.com
```

**Output:**

```text
$ git -C addon-framework remote -v
fork    https://github.com/ssijbabu/addon-framework.git (fetch)
origin  https://github.com/open-cluster-management-io/addon-framework.git (fetch)
$ git -C addon-framework branch --show-current
spire
```

Also set the git identity in the repos that have none (it was missing in `api`, `ocm` and
`clusteradm` during the Azure rebase):

```shell
for r in api sdk-go ocm clusteradm; do
  git -C $r config user.name ssijbabu; git -C $r config user.email ssijbabu@gmail.com
done
```

## Task 3: check the Go wrapper

**Input:** `plan/tools/dgo`.

**Steps:**

```shell
export PATH=/home/ssijbabu/Documents/repos/ocm/plan/tools:$PATH   # add to ~/.bashrc
cd /home/ssijbabu/Documents/repos/ocm/ocm
dgo 'go version && go env GOMODCACHE && id -u'
```

**Output:**

```text
go version go1.27.3 linux/amd64
/gomod
1000
```

Files created through `dgo` are owned by you (uid 1000), not root.

## Task 4: build everything

**Steps:**

```shell
cd /home/ssijbabu/Documents/repos/ocm
for r in api sdk-go ocm clusteradm addon-framework; do
  (cd $r && dgo 'go build ./... && go vet ./...' >/tmp/build-$r.log 2>&1; echo "$r exit $?")
done
```

**Output:**

```text
api exit 0
sdk-go exit 0
ocm exit 0
clusteradm exit 0
addon-framework exit 0
```

Any non-zero exit on clean `main` is an upstream problem. Write it in `plan/questions.md` and
carry on.

## Task 5: save the baseline test results

**Input:** clean builds.

**Steps:** run each repo's own unit test target, in the background, and keep the logs.

```shell
cd /home/ssijbabu/Documents/repos/ocm
mkdir -p plan/baseline
(cd api && dgo 'go test ./... 2>&1' > ../plan/baseline/api.log)
(cd sdk-go && dgo 'make test 2>&1' > ../plan/baseline/sdk-go.log)
(cd ocm && dgo 'make test-unit 2>&1' > ../plan/baseline/ocm.log)
(cd clusteradm && dgo 'make test 2>&1' > ../plan/baseline/clusteradm.log)
(cd addon-framework && dgo 'make test 2>&1' > ../plan/baseline/addon-framework.log)
grep -hE '^(FAIL|--- FAIL)' plan/baseline/*.log | sort | uniq > plan/baseline/known-failures.txt
wc -l plan/baseline/known-failures.txt
```

**Output:** one log per repo, and `known-failures.txt`.

Expected known failure: `api/test/integration/api` fails under plain `go test` because it needs
etcd from envtest. It passes with `make test-api-integration`. Note it as "needs envtest", not as a
real failure.

## Task 6: write down open questions

**Input:** the proposal's Implementation Details and Open Questions.

**Output:** `plan/questions.md`, for example:

```markdown
# Questions

- [ ] Q1 (proposal OQ4): which SPIRE topology for e2e? Decide by Day 26.
- [ ] Q2: does envtest in ocm run Kubernetes >= 1.30? Needed for ValidatingAdmissionPolicy on Day 19.
- [ ] Q3: exact name of the operator ServiceAccount in hosted mode, for the admission policy.
```

## End of day

| Output | Check |
| --- | --- |
| All five repos on `spire`, clean | `git status --short` is empty in each |
| `addon-framework` cloned with a `fork` remote | `git -C addon-framework remote` shows `fork` |
| Baseline logs | `plan/baseline/*.log` |
| Questions | `plan/questions.md` |

No commits today.
