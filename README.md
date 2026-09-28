# sdlc

Personal store for reusable CICD / SDLC / deployment assets: GitHub Actions
workflows, copyable templates, and the docs that go with them. Copy what you
need into a project, or reference it remotely.

## Directory layout

```
.github/workflows/build-deploy.yml       Reusable workflow (on: workflow_call), referenced remotely
templates/pr-build-deploy/
  pr-build-deploy.yml                     Caller workflow a consumer copies into their repo
  docker-compose.yml                      Compose contract stub the runner uses (image: ${IMAGE})
docs/pipeline.md                          How the build/deploy pipeline works and its trade-offs
```

## How to consume

The build/deploy pipeline (see [docs/pipeline.md](docs/pipeline.md) for the full walkthrough):

1. **Reference the reusable workflow remotely** (recommended) so you track updates here:
   ```yaml
   uses: apples723/sdlc/.github/workflows/build-deploy.yml@main
   ```
   Or copy `.github/workflows/build-deploy.yml` into your repo if you want to pin/fork it.
2. **Copy `templates/pr-build-deploy/pr-build-deploy.yml`** into your target repo's
   `.github/workflows/` and retarget its base branch (the template ships with
   `branches: [master]` and a comment marking the line to change to your default
   branch, e.g. `main`).
3. **Copy `templates/pr-build-deploy/docker-compose.yml`** to the runner's
   `compose-dir` (default `/srv/app`). The deploy job injects `IMAGE`.

## Manual setup (done by hand, not automatable)

- Register a **self-hosted runner** with a label matching the `runs-on` input (default `self-hosted`); install Docker Engine + Compose v2 and add the runner user to the `docker` group.
- Create the **`DEPLOY_ENABLED`** repository variable set to `true` in the consuming repo (Settings > Secrets and variables > Actions > Variables). The caller skips itself while unset.
- Create the **compose dir** on the runner (default `/srv/app`) containing the copied `docker-compose.yml`.
- Add a **`Dockerfile`** at the root of the repo being built.

No secrets required: GHCR auth uses the built-in `GITHUB_TOKEN`.
