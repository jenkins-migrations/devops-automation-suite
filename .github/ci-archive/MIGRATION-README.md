# Jenkins to GitHub Actions migration report

## Summary

| Jenkins source | Archived source | GitHub Actions workflow |
| --- | --- | --- |
| `Jenkinsfile` | `.github/ci-archive/Jenkinsfile` | `.github/workflows/deployment.yml` |
| `simpledeclarative/Jenkinsfile` | `.github/ci-archive/simpledeclarative/Jenkinsfile` | `.github/workflows/simple-declarative.yml` |
| `zomatoanypoint/Jenkinsfile` | `.github/ci-archive/zomatoanypoint/Jenkinsfile` | `.github/workflows/zomato-anypoint.yml` |

All three sources are declarative Jenkins pipelines. They contain no shared-library,
scripted-pipeline, YAML-pipeline, scheduled-trigger, or source-control-trigger
references, so no shared-library expansion was required and the replacement workflows
use `workflow_dispatch`.

## Pipeline mappings

- `agent any` maps to GitHub-hosted `ubuntu-latest` runners.
- Jenkins stages map to ordered steps or jobs.
- Jenkins parameters map to typed `workflow_dispatch` inputs.
- The staging and production `input` gates map to the `staging` and `production`
  GitHub environments. Configure required reviewers on both environments. GitHub
  environment approvals record the reviewer but cannot collect the Jenkins `PROCEED`
  or `DEPLOYMENT_TYPE` values, so rejecting an environment replaces `PROCEED`, and
  `deployment_type` is collected when the workflow starts.
- Jenkins' 5-minute staging and 60-minute production approval timeouts have no exact
  GitHub Actions equivalent. Configure environment protection rules according to the
  repository's approval policy.
- `publishTestResults` maps to a pinned `actions/upload-artifact` step that retains
  Surefire XML reports.
- Jenkins post conditions map to the `post` job in `deployment.yml`. GitHub Actions
  provides job results but these workflows do not configure an external notifier.

## Required configuration

Create the `dev`, `staging`, and `production` GitHub environments. Store a base64-encoded
kubeconfig in an environment secret named `KUBE_CONFIG` for each environment. Protect
the staging and production environments with required reviewers. The runner must be
authorized to reach each Kubernetes API server.

Create these Actions secrets for `zomato-anypoint.yml`:

| Secret | Jenkins source |
| --- | --- |
| `ANYPOINT_USERNAME` | Username from `anypoint.credentials` |
| `ANYPOINT_PASSWORD` | Password from `anypoint.credentials` |

The archived AnyPoint pipeline redacted the password property as
`-Danypoint.******`. The migration uses the standard `anypoint.password` Maven property;
confirm that property against the application's Mule Maven plugin configuration before
the first deployment.

No repository variables are required. Each workflow grants only `contents: read`.
External actions are from GitHub's verified `actions` organization and are pinned to
full commit SHAs:

| Action | Version |
| --- | --- |
| `actions/checkout` | `v7.0.1` |
| `actions/setup-java` | `v6.0.0` |
| `actions/upload-artifact` | `v7.0.1` |

## Manual review and validation

This repository contains pipeline examples only: it has no Maven `pom.xml`, application
source, Kubernetes manifests, or smoke-test service. Consequently, application builds
and deployments cannot be executed from this repository until those inputs are added
or the workflows are moved to the corresponding application repositories. The
placeholder `*.example.com` health endpoints must also be replaced before deployment.

The migration should be validated with:

```text
actionlint .github/workflows/*.yml
git diff --check
```

Rollback consists of restoring the archived Jenkinsfiles to their original paths and
disabling the replacement workflows. The archive is reference material and is not
executed by GitHub Actions.
