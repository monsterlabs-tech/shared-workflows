# shared-workflows

Source of truth for reusable GitHub Actions workflows and composite actions used across `monsterlabs-tech/*` repos.

Lives next to `DevOps/` (not inside it) because it is consumed by every service repo, not just infra. Versioned via git tags; consumers pin by tag.

## Layout

```
.github/
  workflows/         reusable workflows (callable via `uses: monsterlabs-tech/shared-workflows/.github/workflows/<name>.yml@<tag>`)
  actions/           composite actions (callable via `uses: monsterlabs-tech/shared-workflows/.github/actions/<name>@<tag>`)
```

## Reusable workflows

| Workflow | Purpose |
|---|---|
| `terraform-plan.yml` | `terraform init + plan` with PR comment. Used by infra repos. |
| `terraform-apply.yml` | `terraform init + apply -auto-approve`. Gated on env protection rules. |
| `docker-build-push.yml` | Build image, push to GHCR with metadata-action tags. |
| `ecs-deploy.yml` | Render task definition with new image, `aws ecs update-service`, wait for stable. Replaces the legacy SSH-to-EC2 deploy in admin-api. |
| `jira-validate.yml` | Validate PR title contains a JIRA ticket and that the ticket exists. |
| `image-cleanup.yml` | Keep the most recent N GHCR image versions for a package; delete the rest. |

## Composite actions

| Action | Purpose |
|---|---|
| `jira-check` | Underlying validation logic for `jira-validate.yml`. Direct usage permitted for non-PR contexts. |
| `setup-tools` | Pin `terraform`, `aws-cli`, `jq`, `gh` versions for reproducible runs. |

## Consumption pattern

Service repos add a thin caller workflow:

```yaml
# in <service-repo>/.github/workflows/deploy-dev.yml
jobs:
  deploy:
    uses: monsterlabs-tech/shared-workflows/.github/workflows/ecs-deploy.yml@v1
    with:
      environment: dev
      aws-region: eu-west-2
      cluster: maxim-dev-platform
      service: admin-api
      task-definition: admin-api
      container-name: admin-api
      image: ghcr.io/monsterlabs-tech/admin-api:${{ github.sha }}
      role-to-assume: ${{ vars.MAXIM_DEV_DEPLOY_ROLE }}
```

Always pin to a tag (`@v1`, `@v1.3.0`) — never `@main`. Bumping the pin is an intentional change tracked per repo.

## Versioning

- Tags follow semver: `v1`, `v1.0.0`, `v1.1.0`.
- `v1` is a moving alias to the latest `v1.x.x` (cut by `release.yml` here).
- Breaking changes to inputs => bump major. Consumers pin by major in steady state.

## Org-level config (shared across all consuming repos)

Configure ONCE at `monsterlabs-tech` org level (Settings -> Secrets and variables -> Actions):

**Variables:**
- `AWS_REGION = eu-west-2`
- `MAXIM_DEV_DEPLOY_ROLE  = arn:aws:iam::722448937891:role/maxim-dev-terraform-deploy`
- `MAXIM_DEV_PLAN_ROLE    = arn:aws:iam::722448937891:role/maxim-dev-terraform-plan`
- `MAXIM_PROD_DEPLOY_ROLE = arn:aws:iam::319627300689:role/maxim-prod-terraform-deploy`
- `MAXIM_PROD_PLAN_ROLE   = arn:aws:iam::319627300689:role/maxim-prod-terraform-plan`
- `JIRA_BASE_URL`, `JIRA_PROJECT_KEY`

**Secrets:**
- `JIRA_USER_EMAIL`, `JIRA_API_TOKEN`
- `RELEASE_PAT` (classic PAT with `repo`, `workflow`, `write:packages`)

Per-repo overrides only when the repo genuinely differs. Drift between repos is the failure mode this design is trying to prevent.

## Onboarding a new repo

See `DevOps/docs/service-cicd-template.md` for the full per-service checklist.

Quick version:
1. Add the repo name to `var.github_repos_with_deploy_access` in `DevOps/account-foundations/environments/maxim-{dev,prod}/terraform.tfvars`.
2. `terraform apply account-foundations` for both environments — this rewrites the OIDC trust policy.
3. Drop a caller workflow in the new repo that `uses:` the relevant reusable workflow here.
