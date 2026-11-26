# github-workflows

Reusable GitHub Actions workflows for Docker apps: one way to test, release and
promote images, shared by every app repo.

| Workflow | Trigger (in the caller) | What it does |
|---|---|---|
| `docker-ci.yml` | PRs and pushes to `main` | actionlint, hadolint, `compose config`, builds every image, runs `scripts/ci-test.sh` and `scripts/smoke-test.sh` against the built images |
| `docker-candidate.yml` | `vX.Y.Z-rc.N` tag on main | native amd64/arm64 builds → `ghcr.io/<owner>/<repo>/<name>:vX.Y.Z-rc.N` + provenance attestation → smoke test on every platform against the published images → Trivy → GitHub prerelease with `images.json` |
| `docker-promote.yml` | manual | *(coming next)* promote a tested RC digest to `vX.Y.Z` + `latest` without rebuilding |

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

## Conventions

- External Actions are pinned to commit SHAs, tool images to digests.
  Dependabot bumps the Actions; tool image digests are bumped by hand.
- Callers pin this repo to a release tag (`@v1.0.0`). Tags are immutable.
- Permissions default to `contents: read`; jobs ask for more only when needed.

## License

MIT
