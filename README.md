# pipeline-workflows

Reusable GitHub Actions workflows.

## Publish NuGet Package

See [.github/workflows/publish-nuget.yml](.github/workflows/publish-nuget.yml).

## Publish Docker image to GHCR

Workflow: [.github/workflows/publish-ghcr-image.yml](.github/workflows/publish-ghcr-image.yml).

Builds a Docker image and pushes to `ghcr.io/<repo-owner>/<image_name>` with tags **`<7-char-git-sha>`** and **`latest`**. No version bump or commits to the repo. During `dotnet restore`, authentication against **GitHub Packages (NuGet)** uses a temporary `NuGet.config` passed as a **BuildKit secret** (`nuget_auth`), so the token is not baked into image layers.

**Output:** `git_sha_short` — first 7 characters of the commit SHA used as the immutable image tag.

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
