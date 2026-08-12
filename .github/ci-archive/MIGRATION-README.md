# Jenkins → GitHub Actions Migration Report

This directory archives the original Jenkins pipeline definitions that were migrated to
GitHub Actions workflows. The archived files are kept for reference only and are no
longer used by any CI system.

## 1. Summary

| Original Jenkins file | Archived as | New GitHub Actions workflow | Pipeline type |
| --- | --- | --- | --- |
| `Jenkinsfile` | `.github/ci-archive/Jenkinsfile` | `.github/workflows/deployment-pipeline.yml` | Declarative (parameters + `input` approvals) |
| `simpledeclarative/Jenkinsfile` | `.github/ci-archive/simpledeclarative-Jenkinsfile` | `.github/workflows/simple-declarative.yml` | Declarative |
| `zomatoanypoint/Jenkinsfile` | `.github/ci-archive/zomatoanypoint-Jenkinsfile` | `.github/workflows/zomato-anypoint.yml` | Declarative (credential binding) |

No Jenkins shared libraries (`@Library` / `vars/` functions) were referenced by these
pipelines, so no inline shared-library expansion was required.

## 2. Conversion mapping

| Jenkins construct | GitHub Actions equivalent |
| --- | --- |
| `pipeline { }` | Workflow file under `.github/workflows/` |
| `agent any` | `runs-on: ubuntu-latest` |
| `stages { stage('X') { ... } }` | `jobs:` with `needs:` to preserve sequential order |
| `steps { sh '...' }` | `steps: - run: ...` |
| `echo "..."` | `run: echo "..."` |
| `environment { }` | `env:` at workflow/job/step level |
| `credentials('id')` (`_USR` / `_PSW`) | `secrets.*` mapped to `env:` variables |
| `parameters { choice/booleanParam/string }` | `on.workflow_dispatch.inputs` (`choice`, `boolean`, `string`) |
| `when { expression { ... } }` | job-level `if:` conditions |
| `when { not { params.SKIP_TESTS } }` | `if: ${{ !inputs.skip_tests }}` |
| `when { anyOf { ... } }` | `if:` with `||` |
| `input message: ...` (manual approval) | GitHub **Environment** protection rules (required reviewers) |
| `timeout(time: N, unit: 'MINUTES')` around `input` | Environment **wait timer** / reviewer timeout |
| `publishTestResults` | `actions/upload-artifact` publishing `**/surefire-reports/*.xml` |
| `post { success { } }` | Final `notify` job with `if: always()` + `!contains(needs.*.result, 'failure')` |
| `post { failure { } }` | Final `notify` job step with `if: contains(needs.*.result, 'failure')` |

## 3. Required secrets

Configure these in **Settings → Secrets and variables → Actions**:

| Secret | Used by | Source in Jenkins |
| --- | --- | --- |
| `ANYPOINT_CREDENTIALS_USR` | `zomato-anypoint.yml` | Username half of the `anypoint.credentials` username/password credential |
| `ANYPOINT_CREDENTIALS_PSW` | `zomato-anypoint.yml` | Password half of the `anypoint.credentials` username/password credential |

No repository variables are required.

## 4. Required environments

`deployment-pipeline.yml` replaces the Jenkins `input` approval gates with GitHub
Environments. Create these under **Settings → Environments** and add **required
reviewers** so that deployments pause for manual approval:

| Environment | Replaces | Suggested protection |
| --- | --- | --- |
| `staging` | `stage('Approval for Staging')` — `input`, 5 minute timeout | Required reviewers |
| `production` | `stage('Production Approval')` — `input`, 60 minute timeout, `submitterParameter: 'APPROVER'` | Required reviewers (approver is recorded on the deployment) |

## 5. Manual review items / behaviour differences

1. **Approval parameters.** Jenkins' `input` step could collect parameters at approval
   time (`PROCEED`, `DEPLOYMENT_TYPE`). GitHub Environment approvals cannot collect
   input, so `DEPLOYMENT_TYPE` is now the `deployment_type` `workflow_dispatch` input
   supplied when the run is started. The staging `PROCEED` boolean is redundant and is
   represented by approving/rejecting the environment.
2. **Approver identity.** `submitterParameter: 'APPROVER'` has no direct expression
   equivalent; the approving user is recorded in the environment's deployment history.
3. **Approval timeouts.** Jenkins' `timeout(...)` around `input` maps to environment
   wait timers / GitHub's own approval expiry (30 days) rather than an exact 5- or
   60-minute window.
4. **Masked password argument.** The archived `zomatoanypoint/Jenkinsfile` contained a
   masked `-Danypoint.******` argument. It has been restored as the Maven
   `anypoint.password` property, populated from `ANYPOINT_CREDENTIALS_PSW`.
5. **Toolchain.** Jenkins agents provided Maven, JDK and `kubectl` implicitly. The
   workflows now set up a JDK explicitly with `actions/setup-java`; `kubectl` (and any
   cluster credentials/kubeconfig) must be provisioned on the runner or added as a
   setup step before the `kubectl apply` commands will succeed.
6. **Triggers.** Jenkins jobs were triggered externally (no `triggers` block). The
   build/test workflows now run on `push`, `pull_request` and `workflow_dispatch`; the
   deployment pipeline is `workflow_dispatch` only, matching its parameterised nature.

## 6. Security notes

- All third-party actions are pinned to full commit SHAs with a version comment.
- Only actions from the verified `actions` organisation are used.
- Each workflow declares a least-privilege `permissions: contents: read` block.
- Credentials are read from GitHub Secrets and passed via environment variables; they
  are never interpolated directly into `run:` script bodies.

## 7. Validation

All generated workflows were validated with `actionlint` (v1.7.12, including the
bundled shellcheck checks) and reported no errors.
