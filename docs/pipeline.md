# Reusable build and deploy pipeline

Two workflows. `.github/workflows/build-deploy.yml` is reusable (`on: workflow_call`): its `build` job builds the repository's `Dockerfile` and pushes `ghcr.io/<owner>/<repo>:pr-<number>` to GHCR, then its `deploy` job runs `docker compose pull`, `docker compose down`, `docker compose up -d` on a self-hosted runner with `IMAGE` set to the pushed tag. `templates/pr-build-deploy/pr-build-deploy.yml` is the caller: a consumer copies it into their own repo's `.github/workflows/`, retargets its base branch to their default branch, and it runs on pull requests gated on the repository variable `DEPLOY_ENABLED`.

It must be called from a `pull_request` event: the image tag comes from the pull request number, and the `build` job fails with an explicit error if there is none.

The `deploy` job only runs for pull requests from branches in this repository. Fork pull requests still build (so the `Dockerfile` is validated) but do not push or deploy, because GitHub hands a fork's `pull_request` run a read-only `GITHUB_TOKEN` that cannot write to GHCR. Same-repo pull requests do deploy, which means unreviewed branch code runs as a container on your host before anyone approves it. That is the accepted trade for deploy-on-PR in a single-user repository; revisit it before giving anyone else write access, and keep the compose file on the host conservative (no `privileged`, no host networking, minimal volume mounts) since it, not this pipeline, decides how isolated that container is.

Two concurrency groups keep deploys from overlapping. The caller groups runs per pull request, so pushing two commits to one branch deploys them in commit order instead of in whichever order the builds finish. The `deploy` job groups on `runs-on` + `compose-dir`, so different pull requests take turns on the shared compose directory. Neither group cancels a run in progress, because a run cancelled between `down` and `up -d` leaves the host with nothing running.

GitHub keeps only one waiting run per group, so if three runs pile into the same group the middle one is cancelled rather than queued: its image is in GHCR but it never deploys, and the pull request shows a cancelled check. Re-run that check if you want it deployed.

## Consuming it from another repository

Copy `templates/pr-build-deploy/pr-build-deploy.yml` into your target repo's `.github/workflows/` and retarget the base branch (the template ships with `branches: [master]` and a comment marking the line to change to your repo's default branch, e.g. `main`). The caller references the reusable workflow remotely:

```yaml
name: PR build and deploy

on:
  pull_request:
    # Change to your repo's default branch (e.g. main)
    branches: [master]

permissions:
  contents: read
  packages: write

jobs:
  build-deploy:
    uses: apples723/sdlc/.github/workflows/build-deploy.yml@main
    with:
      compose-dir: /srv/app
      runs-on: self-hosted
```

The caller must declare `packages: write`: a reusable workflow inherits its `GITHUB_TOKEN` permissions from the caller. No `secrets: inherit` is needed.

Because this repository is private, each calling repository must be allowed to use its workflows: Settings > Actions > General > Access > "Accessible from repositories owned by ...". This is available on the Team plan.

Give each calling repository its own `compose-dir`. Concurrency groups are scoped to the repository that runs them, so two repositories pointed at the same directory on the same runner are not serialized against each other despite the group name being built from `runs-on` and `compose-dir` - they would take turns on the runner and `down` each other's stack.

## Compose file contract

The `docker-compose.yml` in `compose-dir` on the runner must take the image from the `IMAGE` environment variable that the deploy job sets (see `templates/pr-build-deploy/docker-compose.yml`):

```yaml
services:
  app:
    image: ${IMAGE}
    restart: unless-stopped
    ports:
      - '8080:8080'
```

## Manual setup

None of the following can be provisioned by automation; do them by hand once.

- Register a self-hosted runner on the repository with a label matching the `runs-on` input (default `self-hosted`). Install Docker Engine and Compose v2 on it and add the runner user to the `docker` group.
- Create the compose directory on the runner (default `/srv/app`) containing the `docker-compose.yml` above.
- Add a `Dockerfile` at the root of the repository being built.
- In the consuming repository: create the repository variable `DEPLOY_ENABLED` with value `true` (Settings > Secrets and variables > Actions > Variables). The caller skips itself while it is unset, so you can copy the template into a repo that has no `Dockerfile` or runner yet without failing checks.
- Occasionally prune old images on the runner: `docker image prune -a`. Each pull request leaves a `pr-<number>` image behind and nothing removes them.

No secrets are required: GHCR authentication uses the built-in `GITHUB_TOKEN`.

## Not included

- No environments or approval gates, so nothing depends on Team-plan protection rules.
- No deploy on merge to the default branch.
- No build cache, image scanning, rollback, or multi-arch builds.
- No health check after `up -d`. A green deploy means Compose started the containers, not that the application is serving.
- No teardown when a pull request closes. The last deployed stack keeps running until the next deploy replaces it; run `docker compose down` in the compose directory by hand if you want the host idle. This is deliberate: an automatic teardown on close would tear down whichever pull request happens to be deployed, not necessarily the one that closed.
- No retry or rollback around the requested `pull` -> `down` -> `up -d` ordering. A failed `pull` aborts before anything is torn down and leaves the old stack serving, but a failed `up -d` happens after `down`, so the host is left with nothing running. Re-run the job or run `docker compose up -d` on the host.
