# github-workflows

Reusable GitHub Actions workflows for Docker apps: one way to test, release and
promote images, shared by every app repo.

| Workflow | Trigger (in the caller) | What it does |
|---|---|---|
| `docker-ci.yml` | PRs and pushes to `main` | actionlint, hadolint, `compose config`, builds every image, runs `scripts/ci-test.sh` and `scripts/smoke-test.sh` against the built images |
| `docker-candidate.yml` | `vX.Y.Z-rc.N` tag on main | native amd64/arm64 builds → `ghcr.io/<owner>/<repo>/<name>:vX.Y.Z-rc.N` + provenance attestation → smoke test on every platform against the published images → Trivy → GitHub prerelease with `images.json` → moves the `stg` tag |
| `docker-channel.yml` | manual | points `stg` or `stable` at an already released version (rollback, seeding); verifies release, digests and provenance first |
| `docker-promote.yml` | manual (`workflow_dispatch`) | checks the RC (prerelease exists, its run passed, digests unmoved, provenance verifies, version unreleased), then re-tags the **same digests** `vX.Y.Z` (+ `latest` and `stable` if newest) and publishes the stable release with `images.json`. `dry-run` = checks only |

## Using `docker-ci.yml`

```yaml
# .github/workflows/ci.yml in the app repo
name: ci
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  docker:
    uses: christancho/github-workflows/.github/workflows/docker-ci.yml@v1.0.0
    with:
      images: '[{"name":"api","context":".","dockerfile":"api/Dockerfile"}]'
      compose-files: docker-compose.yml
```

Inputs:

| Input | Default | Notes |
|---|---|---|
| `images` | (required) | JSON list: `name`, `context`, `dockerfile` (relative to `context`) |
| `compose-files` | `""` | validated with `.env.example` as env file when present |
| `test-script` | `scripts/ci-test.sh` | skipped if missing |
| `smoke-script` | `scripts/smoke-test.sh` | skipped if missing |
| `runner` | `ubuntu-latest` | keep GitHub-hosted for public repos: fork PRs run here |

Both scripts get `IMAGE_<NAME>` (upper-cased, `-` → `_`) pointing at the image
built in this run, so they test the artifact, not the source tree. The smoke
script should be written so it also works with published images; release
candidates reuse it.

## Using `docker-candidate.yml`

```yaml
# .github/workflows/release-candidate.yml
name: release-candidate
on:
  push:
    tags: ["v*-rc.*"]
permissions:
  contents: write
  packages: write
  id-token: write
  attestations: write
jobs:
  candidate:
    uses: christancho/github-workflows/.github/workflows/docker-candidate.yml@v1.1.0
    with:
      images: '[{"name":"api","context":".","dockerfile":"api/Dockerfile"}]'
```

Cut an RC: `git tag v1.2.0-rc.1 origin/main && git push origin v1.2.0-rc.1`.
Verify an image: `gh attestation verify oci://<image>@<digest> -R <owner>/<repo> --signer-repo christancho/github-workflows`
(the signer is this repo, because the build runs in the reusable workflow).

## Using `docker-promote.yml`

```yaml
# .github/workflows/promote.yml
name: promote
on:
  workflow_dispatch:
    inputs:
      candidate: { description: "RC to promote, e.g. v1.4.0-rc.2", required: true, type: string }
      dry-run: { description: "Only run the checks", type: boolean, default: false }
permissions:
  contents: write
  packages: write
  attestations: read
jobs:
  promote:
    uses: christancho/github-workflows/.github/workflows/docker-promote.yml@v1.2.0
    with:
      candidate: ${{ inputs.candidate }}
      dry-run: ${{ inputs.dry-run }}
```

Run it: Actions → promote → Run workflow (from `main`), or
`gh workflow run promote.yml -f candidate=v1.4.0-rc.2`. Only admins/maintainers pass the checks.

## Channels

| Tag | Moves when | Typical user |
|---|---|---|
| `stg` | an RC passes (docker-candidate) or by hand (docker-channel) | the maintainer's own instance |
| `stable` / `latest` | promote (newest stable only) or by hand (docker-channel) | people installing the app |
| `vX.Y.Z`, `vX.Y.Z-rc.N` | never (immutable) | pinning, dev stacks |

Caller for rollback:

```yaml
# .github/workflows/channel.yml
name: channel
on:
  workflow_dispatch:
    inputs:
      version: { description: "Released version, e.g. v1.0.0 or v1.1.0-rc.2", required: true, type: string }
      channel: { type: choice, options: [stg, stable], default: stg }
permissions:
  contents: read
  packages: write
  attestations: read
jobs:
  channel:
    uses: christancho/github-workflows/.github/workflows/docker-channel.yml@v1.3.0
    with:
      version: ${{ inputs.version }}
      channel: ${{ inputs.channel }}
```

## Conventions

- External Actions are pinned to commit SHAs, tool images to digests.
  Dependabot bumps the Actions; tool image digests are bumped by hand.
- Callers pin this repo to a release tag (`@v1.0.0`). Tags are immutable.
- Permissions default to `contents: read`; jobs ask for more only when needed.

## License

MIT
