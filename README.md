# pipeline-workflows

Reusable GitHub Actions workflows.

## Publish NuGet Package

See [.github/workflows/publish-nuget.yml](.github/workflows/publish-nuget.yml).

## Bump .csproj minor version

Workflow: [.github/workflows/bump-csproj-minor.yml](.github/workflows/bump-csproj-minor.yml).

Increments **minor** in `<Version>` (`MAJOR.MINOR.PATCH` → `MAJOR.(MINOR+1).PATCH`), commits only that `.csproj`, rebases on the current branch, and pushes. No `dotnet build` / pack.

### Caller permissions

```yaml
permissions:
  contents: write
```

### Caller example

```yaml
on:
  workflow_dispatch:

jobs:
  bump:
    uses: davicbaba/pipeline-workflows/.github/workflows/bump-csproj-minor.yml@main
    with:
      project_path: Rental.Api/Rental.Api.csproj
      # optional — default: chore: bump version to {version} [skip ci]
      # commit_message: 'release: v{version}'
    secrets:
      PUBLISH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Optional input **`commit_message`**: use the placeholder `{version}` where the new version should appear.

**Outputs:** `version`, `commit_sha` (full SHA after push; pass to `publish-ghcr-image` as `checkout_ref` so the image builds from the bump commit).

## Publish Docker image to GHCR

Workflow: [.github/workflows/publish-ghcr-image.yml](.github/workflows/publish-ghcr-image.yml).

Builds a Docker image and pushes to `ghcr.io/<repo-owner>/<image_name>` with tags **`<7-char-git-sha>`** and **`latest`**. The short tag is taken from **`git rev-parse HEAD`** after checkout (not `GITHUB_SHA`), so it matches the sources being built. Optional input **`checkout_ref`** (full SHA or ref) selects which commit to build; leave empty to use the workflow’s trigger commit (`github.sha`). During `dotnet restore`, authentication against **GitHub Packages (NuGet)** uses a temporary `NuGet.config` passed as a **BuildKit secret** (`nuget_auth`), so the token is not baked into image layers.

**Output:** `git_sha_short` — first 7 characters of `HEAD` after checkout.

### Caller permissions

```yaml
permissions:
  contents: read
  packages: write
```

`contents: read` is required so `actions/checkout` can clone the caller repository. If you set only `packages: write`, every other permission defaults to **none** and checkout fails with `Repository not found` on private repos. `contents: write` is not required (this workflow does not push commits).

### Caller example

```yaml
jobs:
  publish-image:
    uses: davicbaba/pipeline-workflows/.github/workflows/publish-ghcr-image.yml@main
    with:
      dockerfile_path: Rental.Api/Dockerfile
      context_path: .
      image_name: rental-api
      github_owner: davicbaba
    secrets:
      PUBLISH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Use a **PAT** as `PUBLISH_TOKEN` if `GITHUB_TOKEN` cannot read private packages (e.g. packages in another org). The same value is used for: `docker login` to GHCR, and NuGet password for `nuget.pkg.github.com`.

### Chain after bump-csproj-minor

```yaml
permissions:
  contents: write
  packages: write

jobs:
  bump-version:
    uses: davicbaba/pipeline-workflows/.github/workflows/bump-csproj-minor.yml@main
    with:
      project_path: Rental.Api/Rental.Api.csproj
    secrets:
      PUBLISH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  publish-image:
    needs: bump-version
    uses: davicbaba/pipeline-workflows/.github/workflows/publish-ghcr-image.yml@main
    with:
      dockerfile_path: Rental.Api/Dockerfile
      context_path: .
      image_name: rental-api
      github_owner: davicbaba
      checkout_ref: ${{ needs.bump-version.outputs.commit_sha }}
    secrets:
      PUBLISH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Dockerfile requirement

The **first** `dotnet restore` (after copying only `.csproj` files) must use the secret when present, and fall back to the repo `NuGet.config` for local builds:

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY NuGet.config ./

COPY path/Your.App.csproj path/
RUN --mount=type=secret,id=nuget_auth,required=false \
    bash -c 'if [ -f /run/secrets/nuget_auth ]; then \
      dotnet restore path/Your.App.csproj --configfile /run/secrets/nuget_auth; \
    else \
      dotnet restore path/Your.App.csproj; \
    fi'

COPY path/ path/
RUN dotnet publish path/Your.App.csproj -c Release -o /app/publish /p:UseAppHost=false
# ... runtime stage ...
```

The workflow generates a config with `nuget.org` plus `https://nuget.pkg.github.com/<github_owner>/index.json` and credentials; it does not need to match the `key=` names in your committed `NuGet.config`.

### GHCR visibility

New packages may default to **private**. Adjust visibility under the package settings on GitHub if the image should be public.

## Trigger Dokploy deployment

Workflow: [.github/workflows/trigger-dokploy-deploy.yml](.github/workflows/trigger-dokploy-deploy.yml).

Sends `POST {base}/api/application.deploy` with `x-api-key` and body `{"applicationId":"..."}`.

### Caller example

```yaml
jobs:
  deploy:
    needs: publish-image
    uses: davicbaba/pipeline-workflows/.github/workflows/trigger-dokploy-deploy.yml@main
    with:
      dokploy_base_url: ${{ vars.DOKPLOY_BASE_URL }}
      application_id: ${{ vars.DOKPLOY_APPLICATION_ID }}
    secrets:
      dokploy_api_key: ${{ secrets.DOKPLOY_API_KEY }}
```

Configure in the caller repo: **Variables** `DOKPLOY_BASE_URL`, `DOKPLOY_APPLICATION_ID`; **Secret** `DOKPLOY_API_KEY`.

## Trigger Coolify deployment

Workflow: [.github/workflows/trigger-coolify-deploy.yml](.github/workflows/trigger-coolify-deploy.yml).

Sends `GET {base}/api/v1/deploy?uuid=<uuid>&force=<bool>` with header `Authorization: Bearer <token>`. The token needs the **`deploy`** (or `root`) permission. `resource_uuid` accepts a comma-separated list to deploy several resources at once.

### Caller example

```yaml
jobs:
  deploy:
    needs: publish-image
    uses: davicbaba/pipeline-workflows/.github/workflows/trigger-coolify-deploy.yml@main
    with:
      coolify_base_url: ${{ vars.COOLIFY_BASE_URL }}
      resource_uuid: ${{ vars.COOLIFY_RESOURCE_UUID }}
      # optional: force: true   # rebuild without cache
    secrets:
      coolify_api_token: ${{ secrets.COOLIFY_API_TOKEN }}
```

Configure in the caller repo: **Variables** `COOLIFY_BASE_URL`, `COOLIFY_RESOURCE_UUID`; **Secret** `COOLIFY_API_TOKEN`.

## Deploy Node site to Azure Static Web App

Workflow: [.github/workflows/deploy-node-azure-static-web-app.yml](.github/workflows/deploy-node-azure-static-web-app.yml).

Builds an **npm-based** static site (`npm ci` + build) inside `app_dir` on the GitHub runner, then uploads the already-built output with the official `Azure/static-web-apps-deploy@v1` action (`skip_app_build: true`, so Oryx never runs and you control the Node version). Designed for static frameworks like **Astro**, Vite, Next (static export), etc.

> **Scope:** Node/npm only. A **.NET / Blazor WebAssembly** app can also be hosted on Azure Static Web Apps, but it builds with `dotnet publish` (output `wwwroot`), so it needs a separate dotnet-based workflow — only the build differs, the deploy step is identical. (Blazor **Server** / interactive isn't static-hostable at all.)

### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `app_dir` | yes | — | Directory with the app, relative to repo root (e.g. `Landing`). |
| `output_location` | no | `dist` | Build output dir, relative to `app_dir`. |
| `node_version` | no | `20` | Node.js version used to build. |
| `install_command` | no | `npm ci` | Dependency install command. |
| `build_command` | no | `npm run build` | Build command. |

### Caller example

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'Landing/**'
  workflow_dispatch:

jobs:
  deploy:
    uses: davicbaba/pipeline-workflows/.github/workflows/deploy-azure-static-web-app.yml@main
    with:
      app_dir: Landing
      output_location: dist
    secrets:
      azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
```

Configure in the caller repo: **Secret** `AZURE_STATIC_WEB_APPS_API_TOKEN` — found in **Azure Portal → your Static Web App → Manage deployment token**. No GitHub `permissions` block is required (the workflow only checks out the caller repo and uploads via the token).
