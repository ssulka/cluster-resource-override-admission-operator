# Contributing to Cluster Resource Override Admission Operator

This document covers contribution guidelines for the Cluster Resource Override (CRO) Admission Operator.
The operator deploys and manages the CRO admission webhook, which mutates pod resource requests and limits based on configured ratios.

## Related Resources

| Resource | Link |
|---|---|
| Operator repo (this repo) | [openshift/cluster-resource-override-admission-operator](https://github.com/openshift/cluster-resource-override-admission-operator) |
| Operand repo | [openshift/cluster-resource-override-admission](https://github.com/openshift/cluster-resource-override-admission) |
| CI configuration | [openshift/release/.../cluster-resource-override-admission-operator/](https://github.com/openshift/release/tree/master/ci-operator/config/openshift/cluster-resource-override-admission-operator/) |
| AI guidance | [AGENTS.md](./AGENTS.md) |
| OpenShift docs | [Cluster Resource Override](https://docs.openshift.com/container-platform/latest/nodes/clusters/nodes-cluster-resource-override.html) |

## Review and Approval Policy

Every change in every pull request must be understood and approved by two humans.
This can be the PR author and a reviewer, or — if the author used an AI tool and does not fully understand the contents of the PR — two human reviewers.

**Exception:** PRs authored by deterministic automation tools that are part of our CI and related systems (whose code has been reviewed by the OpenShift engineering org) can be merged with a single human review.

Every change should be closely scrutinized for bugs.
Our software is complex with many interdependencies.
Review changes from multiple angles:

- **Product architecture**: Does this fit the intended design of the CRO operator and OpenShift?
- **Security**: Are there new attack surfaces, credential handling issues, or privilege escalations?
- **Thread safety**: Are shared resources properly synchronized?
- **Regressions**: Could this break existing reconciliation, webhook, or override behavior?
- **Effects on other components**: How does this impact the CRO admission webhook (operand), OLM integration, or cluster admission flow?

## PR Title Convention

If the change tracks a Jira issue, prefix the PR title with the issue key.
This repo uses `AUTOSCALE` for feature work and `OCPBUGS` for bug fixes:

```
AUTOSCALE-895: Add deployment override support
OCPBUGS-99197: Add networking.k8s.io/networkpolicies RBAC to CSV
NO-JIRA: Update Go module dependencies
```

## PR Workflow (Prow / OpenShift CI)

This repo uses [OpenShift CI (Prow)](https://docs.ci.openshift.org/) for continuous integration, not GitHub Actions.
Prow Tide merges PRs once all required tests pass and the correct labels are present.

### Required labels for merge

- `lgtm` — Added by a reviewer via the `/lgtm` command.
- `approved` — Added by an approver listed in the [OWNERS](./OWNERS) file via the `/approve` command.
- `verified` — Added by anyone in the OpenShift org, but typically by the PR author, via the `/verified` command.

### Useful commands

Comment these on the PR:

| Command | Effect |
|---|---|
| `/lgtm` | Add the `lgtm` label after reviewing. In repos using [LGTM mode](https://docs.ci.openshift.org/how-tos/creating-a-pipeline/#the-pipeline-required-command), this also triggers E2E and other second-stage tests. |
| `/lgtm cancel` | Remove the `lgtm` label |
| `/approve` | Add the `approved` label (OWNERS approvers only) |
| `/pipeline required` | Manually trigger all required second-stage tests (e.g., E2Es) without waiting for `/lgtm` |
| `/retest` | Re-run all failed required tests |
| `/retest-required` | Re-run only the failed required tests |
| `/test <job-name>` | Run a specific CI job (job names are defined in [openshift/release](https://github.com/openshift/release/tree/master/ci-operator/config/openshift/cluster-resource-override-admission-operator/)) |
| `/hold` | Prevent the PR from being merged |
| `/hold cancel` | Remove the hold and allow merging |
| `/verified` | Mark the PR as verified |
| `/cherry-pick <branch>` | Create a cherry-pick PR to a release branch |

### Preventing premature merges

- Add the `WIP:` prefix to the PR title (e.g., `WIP: AUTOSCALE-123: Work in progress`).
  Prow adds the `do-not-merge/work-in-progress` label automatically.
- Use `/hold` to temporarily block merging while awaiting additional review or testing.

### LGTM mode and E2E tests

Repos enrolled in [LGTM mode](https://docs.ci.openshift.org/how-tos/creating-a-pipeline/#the-pipeline-required-command) defer second-stage tests (such as E2Es) until the `/lgtm` label is applied.
This avoids wasting CI resources on PRs that haven't been reviewed yet.
If you need to run E2Es before getting `/lgtm` (e.g., to validate before requesting review), use `/pipeline required`.

## Test Expectations

PRs should include tests to verify correctness and prevent future regressions:

- **Unit tests**: Required for new logic, bug fixes, and behavior changes.
  Run with `make unit-test`.
  Unit tests use the Go `testing` package and [testify](https://github.com/stretchr/testify) (`assert` / `require`).
- **E2E tests**: Expected for new features or significant behavior changes.
  E2E tests are in `test/e2e/` and require a running OpenShift cluster.
  Run with `make e2e-local` (requires `KUBECONFIG`, `LOCAL_OPERATOR_IMAGE`, `LOCAL_OPERAND_IMAGE`).

## Verified Label

Use `/verified` to indicate changes have been verified.
Examples:

```
/verified
/verified by e2e tests
/verified by unit tests
/verified later
```

## Generated Code

The following files are generated and should never be hand-edited:

| File(s) | Generator | Regenerate with |
|---|---|---|
| `pkg/apis/operator/v1/zz_generated.deepcopy.go` | `k8s.io/code-generator` | `make codegen` |
| `pkg/apis/autoscaling/v1/zz_generated.deepcopy.go` | `k8s.io/code-generator` | `make codegen` |
| `pkg/generated/**` (clientset, informers, listers) | `k8s.io/code-generator` | `make codegen` |
| `bundle/**` | `hack/generate-bundle.sh` (`operator-sdk generate bundle`) | `make bundle` |

After modifying API types in `pkg/apis/`, regenerate and commit the results in the same PR:

```bash
make codegen  # regenerates deepcopy and typed clients/informers/listers
make bundle   # regenerates OLM bundle manifests
```

## Development Quick Reference

| Task | Command |
|---|---|
| Build operator binary | `make build` |
| Run unit tests | `make unit-test` |
| Build dev image | `make local-image LOCAL_OPERATOR_IMAGE=<image>` |
| Push dev image | `make local-push LOCAL_OPERATOR_IMAGE=<image>` |
| Deploy operator locally (no OLM) | `make deploy-local LOCAL_OPERATOR_IMAGE=<image> LOCAL_OPERAND_IMAGE=<image>` |
| Deploy operator via OLM (local) | `make deploy-olm-local` |
| Undeploy operator (no OLM) | `make undeploy` (alias for `undeploy-local`) |
| Undeploy operator (OLM) | `make undeploy-olm` |
| Run E2E tests (local) | `make e2e-local KUBECONFIG=<path> LOCAL_OPERATOR_IMAGE=<image> LOCAL_OPERAND_IMAGE=<image>` |
| Run E2E tests (CI) | `make e2e-ci` |
| Create example CRO CR | `make create-cro-cr` |
| Create test pod | `make create-test-pod` |
| Regenerate typed clients | `make codegen` |
| Regenerate OLM bundle | `make bundle` |
| Update vendored deps | `make vendor` |
| Build operator registry image | `make operator-registry-image` |

## Pre-Submit Checklist

Before requesting review:

1. `make build` — Verify the code compiles
2. `make unit-test` — Run unit tests
3. `go fmt ./...` — Format code
4. `make codegen && git diff --exit-code` — Ensure generated files are up to date
5. Review your diff for secrets, credentials, or debug code
6. Address any [CodeRabbit](https://coderabbit.ai/) review feedback — as a courtesy to the human reviewer who follows.
   Responding with an explanation of why you're not acting on a suggestion is fine; the goal is to resolve straightforward issues so human reviewers can focus on the substantive aspects.

## Code Style

- Run `gofmt` (or `go fmt ./...`) before committing
- Follow Go conventions for error strings: lowercase, no trailing punctuation, wrap with `fmt.Errorf("context: %w", err)`
- Use structured logging via `klog` (the project standard)
- Match import grouping in the file you are editing; do not rewrite imports to a new house style
- In Markdown files, put each sentence on its own line

## AI Code Review

This repo uses [CodeRabbit](https://coderabbit.ai/) for automated code review.
Contributors should address CodeRabbit feedback before requesting human review.
Responding with an explanation of why you're not acting on a suggestion is fine; the goal is to resolve straightforward issues so human reviewers can focus on the substantive aspects.
