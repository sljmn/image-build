# image-build

Builds app images on Avrea, pushes them to GHCR and deploys them to Dokku with
`git:from-image`. The app servers never build images.

## In an app repository

1. Add the image job at the end of `.github/workflows/ci.yml`, after the test jobs:

   ```yaml
     image:
       needs: [lint, test]          # every test job
       if: github.ref == format('refs/heads/{0}', github.event.repository.default_branch)
       uses: sljmn/image-build/.github/workflows/image.yml@main
       with:
         builder: dockerfile          # or herokuish (match `dokku builder:report`)
       permissions:
         contents: read
         packages: write
         statuses: write
   ```

   Herokuish apps pass their non-secret build-time config as `build-env`
   (one `KEY=value` per line). Secrets never go into the image. When the
   buildpacks are set on the server (`dokku buildpacks:list`) instead of in a
   `.buildpacks` file, pass them as `buildpacks` (one URL per line). Dockerfile
   apps pass `--build-arg` values from `docker-options` as `build-args`.
   `bin/adopt REPO --target dokku@HOST:APP [--builder herokuish]` does steps 1
   and 2.

2. Add `bin/deploy`:

   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   exec bash -c "$(gh api -H 'Accept: application/vnd.github.raw' repos/sljmn/image-build/contents/bin/deploy-image)" deploy-image "$@"
   ```

3. Deploy with `bin/deploy` (pushed HEAD of the default branch) or
   `bin/deploy <sha>` to deploy or roll back to an earlier commit. It waits
   for CI and the image, then runs `dokku git:from-image` over the `dokku` git
   remote (`--remote NAME` for another remote).

## Servers

Each Dokku server is logged in to ghcr.io with a read-only token
(`read:packages`), stored in 1Password vault "Agent VPS", item
"GHCR read token for Dokku servers":
`dokku registry:login [--global] --password-stdin ghcr.io sljmn`
(`--global` exists from Dokku 0.37).

Dokku runs the Procfile `release` task and app.json deploy scripts for image
deploys as it does for git pushes.
