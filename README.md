# workflows

Shared GitHub Actions for goraxe-org repositories.

- `actions/*`: composite actions (`uses: goraxe-org/workflows/actions/<name>@main`).
- `.github/workflows/ci.yaml`: reusable CI workflow (nx affected targets).
- `.github/workflows/build-sign.yaml`: reusable build, push and sign
  workflow for container images (below).
- `.github/workflows/sign.yaml`: reusable sign-only workflow for an image
  you built and pushed yourself
  (see [Sign an image you built yourself](#sign-an-image-you-built-yourself)).

This repository is **public**, so any repository can call its reusable
workflows: public and private goraxe-org repositories, and repositories
under the `goraxe` user account.

## build-sign

Builds a container image with `docker/build-push-action`, pushes it to GHCR,
and signs it with cosign in one of two modes (see [Signing modes](#signing-modes)):

- **keyless** for public repositories: Sigstore Fulcio certificate from the
  job's GitHub OIDC token, entry in the public Rekor log.
- **key** for private repositories: a cosign key pair from secrets and **no
  transparency log**, so nothing about the image reaches Rekor, Fulcio or
  the CT logs.

On every push run it:

1. builds and pushes the image (tags and labels from `docker/metadata-action`);
2. records the pushed digest;
3. `cosign sign <image>@<digest>` (keyless, or with the key), in both modes
   in the legacy cosign format (`.sig`/`.att` tags) that Kyverno's default
   `Cosign` verifier reads;
4. generates an SPDX SBOM with syft (`anchore/sbom-action`) and attaches it
   with `cosign attest --type spdxjson`;
5. attaches SLSA v1 build provenance with `cosign attest --type slsaprovenance1`;
6. verifies the signature and both attestations (keyless: against the run's
   exact signing identity; key: against `cosign-public-key`, skipped with a
   notice when that is empty), and outputs `digest`, `image` and `signing`.

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
      id-token: write # GitHub OIDC: keyless signing, provenance claims
    uses: goraxe-org/workflows/.github/workflows/build-sign.yaml@main
    with:
      context: .
      platforms: linux/amd64
      push: ${{ github.event_name != 'pull_request' }}
      build-args: |
        VERSION=1.2.3
      # signing: auto (default) = key for private repositories, else keyless.
      # Key mode only: the public key, to verify right after signing.
      cosign-public-key: ${{ vars.COSIGN_PUBLIC_KEY }}
    # Key mode only (ignored in keyless mode). Either pass everything:
    #   secrets: inherit
    # or only the two secrets build-sign reads:
    secrets:
      COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
      COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}

  use-it:
    needs: image
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ needs.image.outputs.image }}" # ghcr.io/owner/repo@sha256:...
```

Call it at `@main`. In keyless mode the ref is part of the signing identity
(see [Verifying](#verifying)), so a different ref means a different identity.

The secrets block maps the caller's secrets onto the names build-sign
declares (`COSIGN_PRIVATE_KEY`, `COSIGN_PASSWORD`). If a repository keeps the
key under other names, map those:
`COSIGN_PRIVATE_KEY: ${{ secrets.<your key secret> }}`.

### Permissions

The workflow sets `permissions: {}` at the top and asks for exactly these on
its single job. A reusable workflow cannot get more than its caller grants,
so the **calling job** must grant all three, also on `pull_request` runs
(job permissions are static, even though nothing is pushed or signed there):

| Permission        | Why                                                 |
| ----------------- | --------------------------------------------------- |
| `contents: read`  | checkout                                            |
| `packages: write` | push to ghcr.io, attach signatures and attestations |
| `id-token: write` | GitHub OIDC token: Fulcio (keyless), provenance     |

`id-token: write` is needed in key mode too: job permissions cannot depend
on the signing mode, and the provenance predicate is built from the OIDC
token's claims. In key mode the token is only fetched from GitHub and decoded
in the job; it is never sent to Fulcio or Rekor.

`attestations: write` is **not** needed: provenance is a cosign attestation,
not a GitHub artifact attestation (see below).

### Inputs

| Input               | Default                      | Description                                                    |
| ------------------- | ---------------------------- | -------------------------------------------------------------- |
| `registry`          | `ghcr.io`                    | Registry to push to.                                           |
| `image-name`        | caller's `github.repository` | Image name without the registry. Lowercased.                   |
| `context`           | `.`                          | Build context.                                                 |
| `file`              | `{context}/Dockerfile`       | Dockerfile path.                                               |
| `platforms`         | `linux/amd64`                | Comma-separated platforms. QEMU is set up for anything else.   |
| `tags`              | branch, pr, semver, long sha | `docker/metadata-action` `tags` rules, one per line.           |
| `flavor`            | (empty)                      | `docker/metadata-action` `flavor`.                             |
| `labels`            | (empty)                      | Extra labels, merged with the generated OCI labels.            |
| `build-args`        | (empty)                      | `KEY=value` per line. Recorded in the provenance: no secrets.  |
| `push`              | `true`                       | `false` = build only, no login, push, signing or attestations. |
| `signing`           | `auto`                       | `keyless`, `key`, or `auto` (see below).                       |
| `cosign-public-key` | (empty)                      | Key mode: public key PEM for the post-sign verify. Not secret. |

`signing: auto` resolves to `key` when `github.event.repository.private` is
`true` and to `keyless` when it is `false`. If the event payload does not
carry the repository (some event types), `auto` fails instead of guessing:
set `signing` explicitly there. The resolved mode is the `signing` output.

### Secrets

Only key mode reads them. In key mode a push run fails at its first step,
before building, if either is empty. `push: false` runs (pull requests) do
not need them.

- `COSIGN_PRIVATE_KEY`: the whole `cosign.key` file written by
  `cosign generate-key-pair`, as multi-line PEM, from
  `-----BEGIN ENCRYPTED SIGSTORE PRIVATE KEY-----` to the END line.
- `COSIGN_PASSWORD`: the password that key was generated with, plain text.

The matching `cosign.pub` (`-----BEGIN PUBLIC KEY-----`) is not secret: pass
it as `cosign-public-key` (for example from an Actions variable,
`vars.COSIGN_PUBLIC_KEY`) and put it in the Kyverno policy.

The current goraxe key lives in Vault at `kvv2/cosign/image-signing`
(field `public_key`); its PEM file has sha256
`bab823c288ca28a70f767acc2dee92677ae5bb137b64824bb50621325b79108e`.

Where to store them:

- **goraxe-org repositories**: organization secrets `COSIGN_PRIVATE_KEY` and
  `COSIGN_PASSWORD`, visible to private repositories (or to selected
  repositories only). The public key fits an organization variable.
- **`goraxe/*` (personal account) repositories**: a personal account has no
  organization secrets, so set both as **repository** secrets in each repo.

### Outputs

| Output    | Example                                 |
| --------- | --------------------------------------- |
| `digest`  | `sha256:4f1c...`                        |
| `image`   | `ghcr.io/goraxe-org/foo@sha256:4f1c...` |
| `signing` | `key` or `keyless`                      |

`digest` and `image` are empty when `push` is `false`; `signing` is always set.

### Signing modes

**keyless** signs, and attests the SBOM and provenance, with:

```sh
cosign sign   --yes --use-signing-config=false --new-bundle-format=false <image>@<digest>
cosign attest --yes --use-signing-config=false --new-bundle-format=false \
  --type spdxjson|slsaprovenance1 --predicate <file> <image>@<digest>
```

- Fulcio certificate for the job's OIDC identity and an entry in the
  **public** Rekor log, as with the cosign defaults.
- `--new-bundle-format=false`: write the **legacy** cosign format, a
  `sha256-<digest>.sig` tag and a `sha256-<digest>.att` tag, with the Rekor
  bundle (signed entry timestamp) in each layer's
  `dev.sigstore.cosign/bundle` annotation and the certificate in
  `dev.sigstore.cosign/certificate`. Kyverno verifies it with the default
  `type: Cosign` keyless entry (see [Verifying, Keyless](#keyless)).
- `--use-signing-config=false`: cosign v3 refuses the TUF signing config
  with the legacy format ("must provide --new-bundle-format or --bundle").
  Without it, cosign uses its default `--fulcio-url`
  (`https://fulcio.sigstore.dev`) and `--rekor-url`
  (`https://rekor.sigstore.dev`), checked in the cosign v3.0.6 source.

Why not the cosign v3 default (a Sigstore bundle attached as an OCI
referrer): Kyverno 1.19.1 reads that only with `type: SigstoreBundle`, and
that verifier cannot find the bundle on GHCR. GHCR has no OCI referrers API,
and the referrers fallback index entries cosign writes there are typed
`application/vnd.oci.empty.v1+json`, which Kyverno skips
([kyverno/kyverno#16664](https://github.com/kyverno/kyverno/issues/16664)).

> **Revert when #16664 is fixed:** drop `--use-signing-config=false
--new-bundle-format=false` from the keyless steps (and from the keyless
> verify step), and switch the Kyverno keyless attestors to
> `type: SigstoreBundle`.

**key** signs, and attests the SBOM and provenance, with:

```sh
cosign sign   --yes --key env://COSIGN_PRIVATE_KEY \
  --tlog-upload=false --use-signing-config=false --new-bundle-format=false <image>@<digest>
cosign attest --yes --key env://COSIGN_PRIVATE_KEY \
  --tlog-upload=false --use-signing-config=false --new-bundle-format=false \
  --type spdxjson|slsaprovenance1 --predicate <file> <image>@<digest>
```

- `--tlog-upload=false`: no Rekor entry. cosign v3 marks the flag deprecated
  (it prints "prefer using a --signing-config file with no transparency log
  services") but still honours it.
- `--use-signing-config=false`: v3 otherwise loads the public-good signing
  config from TUF, which lists Rekor, and refuses `--tlog-upload=false`
  next to it.
- `--new-bundle-format=false`: write the **legacy** cosign format, a
  `sha256-<digest>.sig` tag (simplesigning payload) and a
  `sha256-<digest>.att` tag (DSSE envelopes), with no Rekor bundle
  annotation. The v3 default, a Sigstore bundle in an OCI referrer, is
  read by Kyverno only with `type: SigstoreBundle`; Kyverno's default
  `Cosign` verifier (the `keys` + `rekor.ignoreTlog` + `ctlog.ignoreSCT`
  policy below) only reads the `.sig`/`.att` tags. Tested with Kyverno CLI
  1.19.1 against this format.
- The OIDC token is not used for signing: with `--key` and no tlog, cosign contacts
  only the registry (no TUF, Fulcio or Rekor). Checked with cosign v3.0.6 on
  a network with no internet access: sign, both attests and verify
  succeeded; the same sign without `--tlog-upload=false` failed trying to
  POST to `rekor.sigstore.dev`.

If cosign drops `--tlog-upload`, the replacement is a `--signing-config`
file with no `rekorTlogUrls`, which requires the new bundle format, and then
the Kyverno policy needs `type: SigstoreBundle`.

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

#### Keyless

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
  --certificate-github-workflow-repository goraxe-org/<repo> \
  --new-bundle-format=false

cosign verify-attestation --type spdxjson         <same flags> ghcr.io/goraxe-org/<repo>@sha256:<digest>
cosign verify-attestation --type slsaprovenance1  <same flags> ghcr.io/goraxe-org/<repo>@sha256:<digest>
```

`--new-bundle-format=false` reads the legacy `.sig`/`.att` tags that keyless
mode writes. The Rekor inclusion is checked offline against the bundle
annotation.

Add `--certificate-github-workflow-ref refs/heads/main` (or a tag) to accept
only images built from that ref of the calling repository. Private packages
need `docker login ghcr.io` (a token with `read:packages`) first.

Kyverno (`verifyImages`, keyless) equivalent. It is the default
`type: Cosign` verifier (no `type:` field), which reads the legacy tags;
`rekor.url` is required for keyless entries:

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

#### Key

```sh
cosign verify --key cosign.pub \
  --insecure-ignore-tlog=true --new-bundle-format=false \
  ghcr.io/<owner>/<repo>@sha256:<digest>
cosign verify-attestation --type spdxjson        <same flags> ghcr.io/<owner>/<repo>@sha256:<digest>
cosign verify-attestation --type slsaprovenance1 <same flags> ghcr.io/<owner>/<repo>@sha256:<digest>
```

`--insecure-ignore-tlog=true` is required (there is no Rekor entry) and
prints a warning; the check is the signature against the public key.

Kyverno (`verifyImages`, key) equivalent. Both `ignoreTlog` and `ignoreSCT`
are needed: without them Kyverno requires a Rekor entry and an SCT, and
key-mode signatures have neither.

```yaml
- imageReferences:
    - ghcr.io/goraxe-org/<repo>*
  attestors:
    - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEYkt9YRTFbMVFYr5mNfoB4r3hFrIJ
              X8gVjkTyrH61XQBJ3RakPIJv+hCRzgNtsa1kskFYluijKDX7xNL9LYme+Q==
              -----END PUBLIC KEY-----
            rekor:
              ignoreTlog: true
            ctlog:
              ignoreSCT: true
  attestations:
    - type: https://spdx.dev/Document
      attestors:
        - entries:
            - keys:
                publicKeys: |-
                  -----BEGIN PUBLIC KEY-----
                  MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEYkt9YRTFbMVFYr5mNfoB4r3hFrIJ
                  X8gVjkTyrH61XQBJ3RakPIJv+hCRzgNtsa1kskFYluijKDX7xNL9LYme+Q==
                  -----END PUBLIC KEY-----
                rekor:
                  ignoreTlog: true
                ctlog:
                  ignoreSCT: true
```

The key pair is shared by every repository that uses it, so a key-mode
signature proves "signed by a holder of the key", not which repository built
the image. Scope the policy with `imageReferences`.

### Notes

- Keyless signing writes to the **public** Rekor log. Each entry's
  certificate includes the calling repository's name, commit SHA, ref and
  workflow path, also for private repositories. That is why `auto` picks
  key mode for private repositories.
- For multi-platform images, the signature and attestations are on the
  index digest. syft scans the `linux/amd64` image of the index only.
- `pull_request` runs from forks get no `id-token: write`, so the caller
  must not call this workflow from fork PRs.
- cosign is v3.0.6, the same as the repositories that already sign
  (karpenter-provider-vultr, cost-exporter, arc-runner-rust).

## Sign an image you built yourself

`.github/workflows/sign.yaml` signs an image that **your own job** already
built and pushed, and attaches the same SBOM and provenance attestations as
build-sign. Use it when build-sign cannot build the image: self-hosted or
ARC runners, a remote buildx driver, a registry cache, extra
`build-contexts`, and so on. Your build job pushes the image and exports its
digest; a second job calls `sign.yaml` with that digest.

It uses the same signing modes, cosign v3.0.6 flags and legacy `.sig`/`.att`
format as build-sign (see [Signing modes](#signing-modes)), so the same
Kyverno key policy verifies images from either workflow. The steps are copied
from build-sign rather than shared, so build-sign stays exactly as it is.

It:

1. checks `digest` is `sha256:<64 hex>` (a tag is rejected, so a mutable
   reference is never signed) and `image` is a repository with no tag or
   digest under `registry`;
2. resolves `signing` (as build-sign: `auto` fails closed when the event
   does not say whether the repository is private) and, in key mode, fails
   at once if either secret is empty;
3. logs in to `registry` with the job's `github.token` and signs
   `<image>@<digest>`;
4. `sbom: true`: generates an SPDX SBOM of the pushed digest with syft
   (`anchore/sbom-action`) and attaches it with `cosign attest --type spdxjson`;
5. `provenance: true`: attaches SLSA v1 provenance with
   `cosign attest --type slsaprovenance1`;
6. verifies the signature and the attestations it wrote (keyless: the run's
   exact identity; key: `cosign-public-key`, skipped with a notice when that
   is empty), and writes a step summary.

### Caller example

```yaml
on:
  push:
    branches: [main]

permissions: {}

jobs:
  build:
    runs-on: ${{ vars.ARC_RUNNER_RB_BUILD }} # any runner
    permissions:
      contents: read # checkout
      packages: write # push to ghcr.io
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: docker/setup-buildx-action@f87e5991a6d7451dcb8d9637bfbc97413f497069 # v4.4.1
      - uses: docker/login-action@dbcb813823bdd20940b903addbd779551569679f # v4.6.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ github.token }}
      - id: build
        uses: docker/build-push-action@c3c9e263c25d99ce0380d002d59b67737d91b0dc # v7.4.0
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}/app:${{ github.sha }}
          # BuildKit's unsigned attestations off: they turn a single-platform
          # image into an index, and sign.yaml attaches signed ones.
          provenance: false
          sbom: false

  sign:
    needs: build
    permissions:
      contents: read
      packages: write # attach signatures and attestations in ghcr.io
      id-token: write # GitHub OIDC: keyless signing, provenance claims
    uses: goraxe-org/workflows/.github/workflows/sign.yaml@main
    with:
      image: ghcr.io/${{ github.repository }}/app # no tag
      digest: ${{ needs.build.outputs.digest }}
      # JSON: a quoted label, or an array of labels.
      runs-on: ${{ toJSON(vars.ARC_RUNNER_RB_BUILD) }}
      # signing: auto (default) = key for private repositories, else keyless.
      cosign-public-key: |
        -----BEGIN PUBLIC KEY-----
        MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEYkt9YRTFbMVFYr5mNfoB4r3hFrIJ
        X8gVjkTyrH61XQBJ3RakPIJv+hCRzgNtsa1kskFYluijKDX7xNL9LYme+Q==
        -----END PUBLIC KEY-----
    secrets:
      COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
      COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

For several images, give each build job its own `digest` output (or use one
build job per image) and one `sign` job per image: a matrix job's outputs
keep only one leg's value.

`sign.yaml` itself only asks for `packages: write` and `id-token: write` on
its job. `contents: read` in the example is harmless; it needs no checkout.

As with build-sign, the secrets block maps the caller's secrets onto the
names `sign.yaml` declares. goraxe-org repositories can use the organization
secrets. **`goraxe/*` (personal account) repositories have no organization
secrets: set `COSIGN_PRIVATE_KEY` and `COSIGN_PASSWORD` as repository
secrets in each repository.** See [Secrets](#secrets).

### sign inputs

| Input               | Default           | Description                                                          |
| ------------------- | ----------------- | -------------------------------------------------------------------- |
| `image`             | (required)        | Repository with registry, no tag or digest. Lowercased.              |
| `digest`            | (required)        | `sha256:<64 hex>` of the pushed image. Anything else fails.          |
| `signing`           | `auto`            | `keyless`, `key`, or `auto`, as build-sign.                          |
| `cosign-public-key` | (empty)           | Key mode: public key PEM for the post-sign verify. Not secret.       |
| `runs-on`           | `"ubuntu-latest"` | JSON for `runs-on`: `'"label"'`, `'["a","b"]'`, or `toJSON(vars.X)`. |
| `sbom`              | `true`            | Generate and attest an SPDX SBOM.                                    |
| `provenance`        | `true`            | Attest SLSA v1 provenance.                                           |
| `registry`          | `ghcr.io`         | Registry to log in to with `github.token`; `image` must be under it. |

Secrets: `COSIGN_PRIVATE_KEY` and `COSIGN_PASSWORD`, as build-sign: only key
mode reads them, and key mode fails before signing if either is empty.

### sign outputs

| Output      | Example                                  |
| ----------- | ---------------------------------------- |
| `image-ref` | `ghcr.io/goraxe/repo/app@sha256:4f1c...` |
| `signing`   | `key` or `keyless`                       |

### Runner requirements

- **Disk:** syft pulls the image to build the SBOM, so the runner needs free
  disk for the whole image. For large images pick a big runner with
  `runs-on` (an ARC scale set, for example), or set `sbom: false`.
- The runner needs `docker` (for `docker/login-action`; cosign and syft read
  the credentials it writes), `curl`, `jq` and `base64`. GitHub-hosted
  runners have them all.

### Provenance

The predicate has the same shape as build-sign's (built from the job's OIDC
claims, `buildType: https://actions.github.io/buildtypes/workflow/v1`), with
two differences: there is no `externalParameters.build` (sign.yaml does not
know the build inputs), and `runDetails.builder.id` is `sign.yaml`'s
`job_workflow_ref`. `externalParameters.workflow` and `resolvedDependencies`
name the calling workflow and commit, which built the image.

### Verifying sign.yaml signatures

**Key mode** writes exactly what build-sign key mode writes: verify with the
same commands and the same Kyverno `keys` attestor
([Verifying, Key](#key)).

**Keyless** (public callers only: the certificate, with the calling
repository, commit and ref, goes to the public Rekor log) is signed by **this
workflow**, a different identity from build-sign:

```text
https://github.com/goraxe-org/workflows/.github/workflows/sign.yaml@refs/heads/main
```

```sh
cosign verify ghcr.io/<owner>/<repo>@sha256:<digest> \
  --certificate-identity-regexp '^https://github\.com/goraxe-org/workflows/\.github/workflows/sign\.yaml@refs/heads/main$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-github-workflow-repository <owner>/<repo> \
  --new-bundle-format=false
```

Kyverno (`verifyImages`, keyless, default `type: Cosign`):

```yaml
- imageReferences:
    - ghcr.io/<owner>/<repo>*
  attestors:
    - entries:
        - keyless:
            subjectRegExp: ^https://github\.com/goraxe-org/workflows/\.github/workflows/sign\.yaml@refs/heads/main$
            issuer: https://token.actions.githubusercontent.com
            additionalExtensions:
              githubWorkflowRepository: <owner>/<repo>
            rekor:
              url: https://rekor.sigstore.dev
```

To accept images from either workflow, match both:
`^https://github\.com/goraxe-org/workflows/\.github/workflows/(build-sign|sign)\.yaml@refs/heads/main$`.
