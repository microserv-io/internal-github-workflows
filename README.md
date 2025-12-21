# Internal GitHub Workflows

Reusable GitHub Actions workflows for Microserv.io projects.

## Available Workflows

### build-monorepo.yaml

Builds container images for changed apps in a monorepo. Automatically detects which apps have changes and only builds those.

**Inputs:**

| Input | Required | Description |
|-------|----------|-------------|
| `apps_config` | Yes | JSON array of app configurations |
| `registry` | Yes | Container registry URL |
| `image_prefix` | Yes | Image name prefix (e.g., `herrewijnen`) |
| `build_all` | No | Force build all apps regardless of changes |

**Secrets:**

| Secret | Required | Description |
|--------|----------|-------------|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Yes | GCP Workload Identity Provider |
| `GCP_SERVICE_ACCOUNT` | Yes | GCP Service Account email |

**App Configuration Format:**

```json
[
  {
    "name": "web",
    "path": "apps/web",
    "dockerfile": "Dockerfile",
    "build_args": "NODE_ENV=production"
  }
]
```

### build-container.yaml

Builds a single container image. Useful for simple repos or when you need more control.

**Inputs:**

| Input | Required | Description |
|-------|----------|-------------|
| `app_name` | Yes | Name of the app |
| `app_path` | Yes | Path to the app directory |
| `dockerfile` | No | Dockerfile path (default: `Dockerfile`) |
| `registry` | Yes | Container registry URL |
| `image_name` | Yes | Image name (without registry) |
| `build_args` | No | Docker build arguments |
| `platforms` | No | Target platforms (default: `linux/amd64`) |

### detect-changes.yaml

Detects which apps in a monorepo have changed. Used internally by `build-monorepo.yaml`.

### version-changelog.yaml

Automatic semantic versioning and changelog generation based on conventional commits.

**Version Bumping Rules:**
- `feat:` → Minor bump (0.X.0)
- `fix:` → Patch bump (0.0.X)
- `feat!:` or `BREAKING CHANGE:` → Major bump (X.0.0)

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `version_file` | No | `package.json` | File to update (package.json, VERSION, pyproject.toml) |
| `changelog_file` | No | `CHANGELOG.md` | Changelog file path |
| `commit_changelog` | No | `true` | Whether to commit the changes |
| `dry_run` | No | `false` | Only calculate, don't commit |

**Outputs:**

| Output | Description |
|--------|-------------|
| `new_version` | The calculated new version |
| `old_version` | The previous version |
| `bump_type` | The bump type (major, minor, patch, none) |

## Usage Examples

### Monorepo (Multiple Apps)

```yaml
# .github/workflows/build.yaml
name: Build

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    uses: microserv-io/internal-github-workflows/.github/workflows/build-monorepo.yaml@main
    with:
      apps_config: |
        [
          {"name": "web", "path": "apps/web"},
          {"name": "admin", "path": "apps/admin"},
          {"name": "api", "path": "apps/api"},
          {"name": "crawler", "path": "apps/crawler"}
        ]
      registry: europe-west4-docker.pkg.dev/microserv-shared-347c/microserv
      image_prefix: herrewijnen
    secrets:
      GCP_WORKLOAD_IDENTITY_PROVIDER: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      GCP_SERVICE_ACCOUNT: ${{ secrets.GCP_SERVICE_ACCOUNT }}
```

### Single Container

```yaml
# .github/workflows/build.yaml
name: Build

on:
  push:
    branches: [main]

jobs:
  build:
    uses: microserv-io/internal-github-workflows/.github/workflows/build-container.yaml@main
    with:
      app_name: myapp
      app_path: .
      registry: europe-west4-docker.pkg.dev/microserv-shared-347c/microserv
      image_name: myapp/app
    secrets:
      GCP_WORKLOAD_IDENTITY_PROVIDER: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      GCP_SERVICE_ACCOUNT: ${{ secrets.GCP_SERVICE_ACCOUNT }}
```

### Version and Changelog (Trunk-Based)

```yaml
# .github/workflows/release.yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  version:
    uses: microserv-io/internal-github-workflows/.github/workflows/version-changelog.yaml@main
    with:
      version_file: package.json
      changelog_file: CHANGELOG.md

  build:
    needs: version
    if: needs.version.outputs.bump_type != 'none'
    uses: microserv-io/internal-github-workflows/.github/workflows/build-monorepo.yaml@main
    with:
      apps_config: |
        [{"name": "app", "path": "."}]
      registry: europe-west4-docker.pkg.dev/microserv-shared-347c/microserv
      image_prefix: myproject
    secrets:
      GCP_WORKLOAD_IDENTITY_PROVIDER: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      GCP_SERVICE_ACCOUNT: ${{ secrets.GCP_SERVICE_ACCOUNT }}
```

## GCP Setup

### Workload Identity Federation

These workflows use Workload Identity Federation for keyless authentication to GCP.

1. Create a Workload Identity Pool:

```bash
gcloud iam workload-identity-pools create "github-actions" \
  --location="global" \
  --display-name="GitHub Actions"
```

2. Create a Provider:

```bash
gcloud iam workload-identity-pools providers create-oidc "github" \
  --location="global" \
  --workload-identity-pool="github-actions" \
  --display-name="GitHub" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

3. Grant the Service Account access:

```bash
gcloud iam service-accounts add-iam-policy-binding "github-actions@PROJECT.iam.gserviceaccount.com" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-actions/attribute.repository/ORG/REPO"
```

### Required Repository Secrets

| Secret | Value |
|--------|-------|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | `projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-actions/providers/github` |
| `GCP_SERVICE_ACCOUNT` | `github-actions@PROJECT.iam.gserviceaccount.com` |

## Change Detection

The monorepo workflow automatically detects changes by:

1. Comparing changed files against each app's path
2. Building only apps with file changes
3. Triggering all builds if root dependencies change:
   - `package.json`, `pnpm-lock.yaml`, `bun.lockb`
   - `requirements.txt`, `go.mod`, `go.sum`
   - `.dockerignore`, `charts/`

## Image Tags

Built images are tagged with:

- Git SHA (short): `abc1234`
- Branch name or `latest` for main branch
