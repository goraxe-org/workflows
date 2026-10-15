# workflows

Shared GitHub Actions for goraxe-org repositories.

- `actions/*`: composite actions (`uses: goraxe-org/workflows/actions/<name>@main`).
- `.github/workflows/ci.yaml`: reusable CI workflow (nx affected targets).
- `.github/workflows/build-sign.yaml`: reusable build, push and keyless-sign
  workflow for container images (below).

This repository is private with Actions access set to "organization", so
only **private** repositories owned by **goraxe-org** can call its reusable
workflows. Public goraxe-org repositories and repositories under the
`goraxe` user account cannot.

## build-sign

Builds a container image with `docker/build-push-action`, pushes it to GHCR,
and signs it with **cosign keyless** (Sigstore Fulcio certificate from the
job's GitHub OIDC token, entry in the public Rekor log). On every push run it:

1. builds and pushes the image (tags and labels from `docker/metadata-action`);
2. records the pushed digest;
3. `cosign sign --yes <image>@<digest>`;
4. generates an SPDX SBOM with syft (`anchore/sbom-action`) and attaches it
   with `cosign attest --type spdxjson`;
5. attaches SLSA v1 build provenance with `cosign attest --type slsaprovenance1`;
6. verifies the signature and both attestations against the run's exact
   signing identity, and outputs `digest` and `image`.

With `push: false` it only builds: no login, no push, no signing. Use that on
`pull_request` runs.

### Usage

```yaml
on:
  push:
    branches: [main]
    tags: ["*"]
  pull_request: {}

jobs:
  image:
    permissions:
      contents: read # checkout
      packages: write # push to ghcr.io, attach signatures and attestations
      id-token: write # cosign keyless signing
    uses: goraxe-org/workflows/.github/workflows/build-sign.yaml@main
    with:
      context: .
      platforms: linux/amd64
      push: ${{ github.event_name != 'pull_request' }}
      build-args: |
        VERSION=1.2.3

  use-it:
    needs: image
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ needs.image.outputs.image }}" # ghcr.io/owner/repo@sha256:...
```

Call it at `@main`. The ref is part of the signing identity (see
[Verifying](#verifying)), so a different ref means a different identity.

### Permissions

The workflow sets `permissions: {}` at the top and asks for exactly these on
its single job. A reusable workflow cannot get more than its caller grants,
so the **calling job** must grant all three, also on `pull_request` runs
(job permissions are static, even though nothing is pushed or signed there):

| Permission        | Why                                                 |
| ----------------- | --------------------------------------------------- |
| `contents: read`  | checkout                                            |
| `packages: write` | push to ghcr.io, attach signatures and attestations |
| `id-token: write` | GitHub OIDC token for Fulcio (keyless signing)      |

`attestations: write` is **not** needed: provenance is a cosign attestation,
not a GitHub artifact attestation (see below).

### Inputs

| Input        | Default                      | Description                                                    |
| ------------ | ---------------------------- | -------------------------------------------------------------- |
| `registry`   | `ghcr.io`                    | Registry to push to.                                           |
| `image-name` | caller's `github.repository` | Image name without the registry. Lowercased.                   |
| `context`    | `.`                          | Build context.                                                 |
| `file`       | `{context}/Dockerfile`       | Dockerfile path.                                               |
| `platforms`  | `linux/amd64`                | Comma-separated platforms. QEMU is set up for anything else.   |
| `tags`       | branch, pr, semver, long sha | `docker/metadata-action` `tags` rules, one per line.           |
| `flavor`     | (empty)                      | `docker/metadata-action` `flavor`.                             |
| `labels`     | (empty)                      | Extra labels, merged with the generated OCI labels.            |
| `build-args` | (empty)                      | `KEY=value` per line. Recorded in the provenance: no secrets.  |
| `push`       | `true`                       | `false` = build only, no login, push, signing or attestations. |

### Outputs

| Output   | Example                                 |
| -------- | --------------------------------------- |
| `digest` | `sha256:4f1c...`                        |
| `image`  | `ghcr.io/goraxe-org/foo@sha256:4f1c...` |

Both are empty when `push` is `false`.

### Why cosign provenance, not `actions/attest-build-provenance`

GitHub artifact attestations (`actions/attest-build-provenance`,
`actions/attest`) only work in **private** repositories on GitHub Enterprise
Cloud. goraxe-org is on the Team plan and almost every image repository is
private, so the action would fail there. cosign attestations work the same
way for public and private repositories, use the same keyless identity and
Rekor log as the signature, are stored next to the image in GHCR, and are
what Kyverno's `verifyImages` `attestations` checks read.

The predicate is the same SLSA v1 shape that `actions/attest-build-provenance`
produces (`buildType: https://actions.github.io/buildtypes/workflow/v1`,
builder id = this reusable workflow's `job_workflow_ref`), built from the
job's OIDC claims, plus the build inputs (context, Dockerfile, platforms,
build args) under `externalParameters.build`.

BuildKit's own provenance and SBOM attestations are turned off
(`provenance: false`, `sbom: false`): they are unsigned and they turn a
single-platform image into an index.

### Verifying

The Fulcio certificate's identity (SAN) is **this reusable workflow**, not the
caller's workflow:

```text
https://github.com/goraxe-org/workflows/.github/workflows/build-sign.yaml@refs/heads/main
```

Every repository that calls `build-sign` signs with that same identity, so a
verifier must also pin the calling repository, which Fulcio records in the
certificate's GitHub workflow repository extension (OID
1.3.6.1.4.1.57264.1.5):

```sh
cosign verify ghcr.io/goraxe-org/<repo>@sha256:<digest> \
  --certificate-identity-regexp '^https://github\.com/goraxe-org/workflows/\.github/workflows/build-sign\.yaml@refs/heads/main$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-github-workflow-repository goraxe-org/<repo>

cosign verify-attestation --type spdxjson         <same flags> ghcr.io/goraxe-org/<repo>@sha256:<digest>
cosign verify-attestation --type slsaprovenance1  <same flags> ghcr.io/goraxe-org/<repo>@sha256:<digest>
```

Add `--certificate-github-workflow-ref refs/heads/main` (or a tag) to accept
only images built from that ref of the calling repository. Private packages
need `docker login ghcr.io` (a token with `read:packages`) first.

Kyverno (`verifyImages`, keyless) equivalent:

```yaml
- imageReferences:
    - ghcr.io/goraxe-org/<repo>*
  attestors:
    - entries:
        - keyless:
            subjectRegExp: ^https://github\.com/goraxe-org/workflows/\.github/workflows/build-sign\.yaml@refs/heads/main$
            issuer: https://token.actions.githubusercontent.com
            additionalExtensions:
              githubWorkflowRepository: goraxe-org/<repo>
            rekor:
              url: https://rekor.sigstore.dev
```

### Notes

- Keyless signing writes to the **public** Rekor log. Each entry's
  certificate includes the calling repository's name, commit SHA, ref and
  workflow path, also for private repositories.
- For multi-platform images, the signature and attestations are on the
  index digest. syft scans the `linux/amd64` image of the index only.
- `pull_request` runs from forks get no `id-token: write`, so the caller
  must not call this workflow from fork PRs.
- cosign is v3.0.6, the same as the repositories that already sign
  (karpenter-provider-vultr, cost-exporter, arc-runner-rust).
