# ci-templates

Shared deploy pipelines for Marketplace Studio projects. A project's own
`.github/workflows/deploy.yml` calls one of these, pinned to one exact commit
of this repository, and passes its own commands and secrets. Nothing here holds
a secret.

| Workflow | Shape | Secrets each Environment must hold |
| --- | --- | --- |
| `deploy-cloudflare.yml` | Cloudflare Workers | `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` |
| `deploy-vercel.yml` | Vercel | `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID` |
| `deploy-container.yml` | Container image | `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` |

This repository is public because a workflow in another organisation can only
call a reusable workflow from a public repository.

## What a run does

1. **Check the request**, in a job with no Environment and no permissions. It
   refuses an environment other than `staging` or `production`, an unknown
   package manager, and any branch other than the default branch.
2. **Deploy**, in the named Environment, one run per environment at a time:
   check the secrets are set, check out, install from the lockfile, then the
   project's `setup`, `migrate`, `deploy` and `smoke` commands in that order.
   Each step except deploy is skipped when its command is empty.
3. **Record** the deployed commit and the calling pipeline in the run summary.
   The pipeline's `uses:` line names the template commit.

## Who sees the secrets

| Step | Secrets |
| --- | --- |
| Install | none |
| `setup_command` | none |
| `migrate_command` | the shape's secrets |
| `deploy_command` | the shape's secrets |
| `smoke_command` | none |

The job token is read-only and is not left in the checkout. There is no
dependency cache, so no other workflow in the repository can write to what a
deploy installs.

## Setting up a project

In the project's repository, under Settings, Environments:

1. Create `staging` and `production`.
2. Add the shape's secrets to each (table above). Environment secrets, not
   repository secrets, so staging never holds production's.
3. On `production`, set Deployment branches to the default branch only, and add
   required reviewers if a person should approve each release.

For `pnpm`, `package.json` must carry a `packageManager` field. `yarn` needs
Node.js 24 or earlier, because it relies on corepack.

## The contract

These inputs are what a project's `deploy.yml` passes. Every template accepts
all of them, including `migrate_command`, so a project can change shape without
changing which keys it sends.

| Input | Required | Default |
| --- | --- | --- |
| `environment` | yes | |
| `node_version` | no | `"22"` |
| `package_manager` | no | `npm` |
| `setup_command` | no | `""` |
| `migrate_command` | no | `""` |
| `deploy_command` | yes | |
| `smoke_command` | no | `""` |

A project pinned to an old commit keeps running that commit's code, so no
change here reaches it until its pin moves. The contract matters when a pin
moves: the new commit must accept every key the project already sends.

- Adding an optional input is safe.
- Removing or renaming an input, or making one required, breaks every project
  the next time its pin moves. Do it only with a matching change to the file
  generator, in the same release.
- Change all three templates together. Only lines marked `template-specific`
  may differ, and CI enforces that.

## Releasing

1. Merge to `main`. Required: the `actionlint` and `templates in step` checks,
   and a code owner's approval.
2. The release is the merge commit's full 40-character SHA.
3. Check the SHA is on `main` of this repository before pinning it. GitHub
   resolves a commit from any fork of a public repository under this
   repository's name, so a SHA from a fork looks identical in a `uses:` line:

   ```bash
   gh api repos/Marketplace-Studio-Dev/ci-templates/compare/<sha>...main --jq .status
   ```

   `identical` or `ahead` means it is on `main`. Anything else, or an error,
   means do not pin it.
4. Set the SHA as the project's template commit. Each project's pipeline moves
   when a pull request updating its `deploy.yml` merges in its own repository.

## Rules on this repository

- A ruleset on `main`: pull request required, code owner review, dismiss stale
  approvals, approval of the most recent push, the two lint checks required, no
  force push, no deletion.
- Actions pinned to full commit SHAs, with the tag in a comment.
