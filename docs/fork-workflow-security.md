# Fork workflow security

All inherited workflow jobs check both `github.actor` and `github.triggering_actor`
against `gomons`, and require this repository to be `gomons/logseq-og`. This covers
initial execution and reruns. Existing event and job conditions still apply.
These checks skip jobs; they do not remove GitHub's Run workflow button for other
users with write access. A user able to change workflow code can remove a check.

## Signing approval

The Windows, macOS, Android and iOS release jobs reference the `signing`
environment and only run from `refs/heads/gomons`. Desktop, Android and iOS source
checkout uses the workflow commit SHA. The separate `git-ref` input is removed so
an approved run cannot select arbitrary code from another branch.

The environment is configured separately in GitHub, not by merging this PR:

- Required reviewer: `gomons`.
- Self review allowed (the owner must be able to approve their own run).
- Administrator bypass disabled.
- Custom deployment branch policy: branch `gomons` only.

Store signing credentials under **Settings > Environments > signing > Environment
secrets**. Do not store signing credentials as repository or organization secrets:
those are not protected by this environment's approval gate. If credentials already
exist at repository scope, move them before running any workflow. No keys are
created, uploaded or rotated by this change.

| Platform | Environment secret names |
| --- | --- |
| Android | `ANDROID_KEYSTORE`, `ANDROID_KEYSTORE_PASSWORD` |
| macOS | `APPLE_CERTIFICATES_P12`, `APPLE_CERTIFICATES_P12_PASSWORD`, `APPLE_ID_EMAIL`, `APPLE_ID_PASSWORD`, `APPLE_TEAM_ID` |
| Windows | `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET` |
| iOS | `APP_STORE_CONNECT_API_KEY_KEY_ID`, `APP_STORE_CONNECT_API_KEY_ISSUER_ID`, `APP_STORE_CONNECT_API_KEY_KEY`, `MATCH_PASSWORD`, `MATCH_GIT_BASIC_AUTHORIZATION` |

The Android reusable workflow obtains secrets from its own environment; the
parent does not forward signing secrets. Its key is written into the runner's
temporary directory with restrictive permissions, passed via environment
variables rather than shell interpolation, and deleted even after a failed step.

## Other protections

- Build tokens default to `contents: read`; release publication alone receives
  `contents: write`. Docker publishing and repository maintenance have their
  separately scoped permissions.
- Checkout does not persist credentials for later dependency scripts.
- External Actions are pinned to source repository commit SHAs, including the
  nightly publisher. The legacy Cloudflare Pages Actions in Build-Demo and
  Build-Stage could not be resolved and are left unchanged; those owner-only
  workflows do not reference the signing environment. They are outside desktop
  and Android builds and should not receive deployment secrets without review.
- Yarn installs use `--frozen-lockfile`. Install hooks still execute dependency
  code; this is not a complete dependency security audit.
- Jobs have a 60-minute timeout. Dormant Sentry script downloads and telemetry
  credentials are removed from desktop and Android build workflows.

## Limits and first run

Only the owner should approve signing after reviewing the exact commit being
built. Environment approval covers the entire protected job, including third-party
build tools; it does not make compromised dependencies safe. Repository
administrators can still edit environment policies or workflow files.

This change does not configure signing identities or make unsigned distributions.
The existing macOS configuration still names Logseq Inc., the certificate import
is guarded for upstream, and Windows signing still names Logseq's Azure account.
Those require a separate setup change before signed builds work in this fork.
Android requires your own signing key. Existing iOS runner/tooling requirements
are also unchanged.

The changes take effect after merging the PR into `gomons`. This PR does not run
signed builds, publish releases or merge itself.
