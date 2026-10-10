# deploy-workflows

The deploy pipeline our repositories share: one reusable GitHub Actions workflow that
deploys to Railway, and the composite actions it is built from. A product repository
keeps only thin, environment-specific callers; everything that could drift between
repositories lives here.

| Path | What it is | Who uses it |
|---|---|---|
| `.github/workflows/deploy.yml` | Reusable workflow: deploy a set of services to one Railway environment, health check each, write a summary, comment on and label the PR. | Every caller. |
| `.github/actions/railway-deploy` | Composite action: deploy one service, from source or from a pre-built image. | `deploy.yml`; a repository with an unusual job shape may use it directly. |
| `.github/actions/resolve-deploy-inputs` | Composite action: turn the raw `workflow_dispatch` inputs (decorated image tags, "(build from source)") into a clean SHA and image tag. | Callers that offer a manual promotion with image tags. |

The workflow is the unit callers use, not the action. Only a workflow can declare a GitHub
Environment, and the Environment is what gives you required reviewers, environment-scoped
secrets and the deploy history on the repository.

## Calling it

A single-service repository that deploys `main` to staging after its CI passes:

```yaml
name: Deploy → Staging
on:
  workflow_run:
    workflows: [Build]
    types: [completed]
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
  pull-requests: write
concurrency:
  group: deploy-staging
  cancel-in-progress: false
jobs:
  deploy-staging:
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    uses: power-tester/deploy-workflows/.github/workflows/deploy.yml@v1
    with:
      environment: staging
      railway_env: staging
      services: '["app"]'
      service_names: '{"app":"app"}'
      health_checks: '{"app":"/api/health"}'
      sha: ${{ github.event.workflow_run.head_sha || github.sha }}
    secrets:
      RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
```

A server/client monorepo with pre-built images and a manual promotion needs no `services`,
`service_names` or `health_checks`: the defaults are `server` and `client`, named
`<repo>-<service>-<env>` in Railway, probed at `/health` and `/`. See
[taskflow's deploy-staging.yml](https://github.com/power-tester/taskflow/blob/main/.github/workflows/deploy-staging.yml)
for the full shape with `resolve-deploy-inputs`.

### What the caller must provide

In the calling repository, per GitHub Environment (Settings → Environments):

- Secret `RAILWAY_TOKEN`: a Railway **project token** for the matching Railway environment
  (Railway → Project Settings → Tokens). Not an account token.
- Variable `<SERVICE>_URL` for each service you want health checked, e.g. `APP_URL`,
  `SERVER_URL`, `CLIENT_URL`. A service without one is deployed but not probed.
- Variable `APP_URL` or `CLIENT_URL` for the link in the summary and PR comment, unless
  the `app_url` input is set.

And in the workflow: `permissions: pull-requests: write` for the PR comment and label,
`contents: read` for the checkout. The caller's `environment`, `vars` and `secrets` are
what the reusable workflow sees; this repository holds no credentials.

### Inputs

| Input | Default | Meaning |
|---|---|---|
| `environment` | required | GitHub Environment to deploy under. Only the deploy job declares it, so required reviewers gate the deploy alone. |
| `railway_env` | required | Railway environment name. |
| `services` | `["server","client"]` | JSON array of service short names; one matrix leg each. |
| `service_names` | `{}` | JSON map short name → Railway service name. Absent services follow `<project>-<service>-<railway_env>`. |
| `project_slug` | repo name | Prefix for service and image names. |
| `health_checks` | `{"server":"/health","client":"/"}` | JSON map short name → path probed after that service deploys. |
| `health_urls` | `{}` | JSON map short name → base URL, overriding the `<SERVICE>_URL` variable. |
| `max_parallel` | `1` | Services deployed at once. `1` keeps them in series. |
| `sha` | `''` | Commit to build from source. Ignored when `image_tag` is set. |
| `image_tag` | `''` | Clean pre-built image tag (`sha-abc1234`). Blank builds from source. |
| `dockerfile_path_template` | `packages/{0}/Dockerfile` | Where Railway expects the Dockerfile; `{0}` is the service. Only matters for image deploys. |
| `registry` | `ghcr.io` | Registry hosting pre-built images. |
| `app_url` | `''` | Link shown in the summary and PR comment. Falls back to `vars.APP_URL`, then `vars.CLIENT_URL`. |
| `label_colors` | ci/dev/staging/production | JSON map environment → PR label colour. |

Secrets: `RAILWAY_TOKEN` (required).

### Source build or pre-built image

With `image_tag` blank, `railway up` uploads the caller's checkout at `sha` and Railway
builds it (Dockerfile, Railpack or Nixpacks, whatever the service is set to). The upload
honours the caller's `.gitignore`.

With `image_tag` set, the action uploads a one-line Dockerfile (`FROM <image>`) at the
configured Dockerfile path, so Railway pulls the image instead of building. Building and
publishing the images is the caller's job; taskflow's `build-image.yml` is the reference.

## Access

This repository is public, so any repository can call its workflow and actions with no
access setting. It holds no credentials: the Railway token only ever arrives through the
caller's `secrets` mapping, and the URLs and service names come from the caller's inputs
and variables. Keep it that way; if it ever has to go private, set Settings → Actions →
General → Access to "Accessible from repositories in the organization" or callers fail
with "workflow was not found".

## Releasing

Callers pin the major tag:

```yaml
uses: power-tester/deploy-workflows/.github/workflows/deploy.yml@v1
```

`deploy.yml` references its own actions the same way (`…/railway-deploy@v1`). A `uses:`
line cannot take an expression, so the workflow cannot ask for "the actions at my own
commit"; the moving major tag is what keeps the two in step. That makes the **major
version the contract**: a change that breaks callers (a renamed input, a changed default)
gets a new major, and `v1` keeps pointing at the last compatible release.

To release:

```bash
git tag -a v1.3.0 -m "…"           # the exact version
git tag -fa v1 -m "v1 → v1.3.0"     # move the major
git push origin v1.3.0 && git push -f origin v1
```

Dependabot's `github-actions` ecosystem bumps a caller's `@v1` reference when a new
major appears, so callers see breaking changes as pull requests rather than surprises.

## Changing it

- `actionlint` runs on every pull request here. Run it locally before pushing.
- A change that every caller should get goes in `deploy.yml` or an action. A change
  that only one repository wants is a new optional input with a default that
  reproduces today's behaviour, so no caller has to move.
- Watch the environment variable names inside `railway-deploy`: `RAILWAY_ENV` is
  reserved by the Railway CLI to select which Railway *backend* it talks to. The action
  uses `DEPLOY_ENV` for a reason.
