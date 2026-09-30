# Release checklist

## Component-specific tag-based publishing

Each package publishes independently when a matching tag is pushed:

| Package | Tag Pattern | Workflow |
|---------|-------------|----------|
| `@bc-forge/sdk` | `sdk-v*` | [`.github/workflows/publish-sdk.yml`](../.github/workflows/publish-sdk.yml) |
| `@bc-forge/cli` | `cli-v*` | [`.github/workflows/publish-cli.yml`](../.github/workflows/publish-cli.yml) |
| `@bc-forge/react` | `react-v*` | [`.github/workflows/publish-react.yml`](../.github/workflows/publish-react.yml) |
| `@bc-forge/indexer` | `indexer-v*` | [`.github/workflows/publish-indexer.yml`](../.github/workflows/publish-indexer.yml) |

All component workflows use the reusable workflow [`.github/workflows/publish-package.yml`](../.github/workflows/publish-package.yml) which:
- Validates the tag matches the expected prefix
- Installs dependencies with `npm ci`
- Runs the package-specific build and test commands
- Validates tarball contents with `npm run test:tarball`
- Publishes to npm using OIDC trusted publishing (falls back to `NPM_TOKEN` secret if needed)
- Sets `npm config set provenance true` for provenance attestations

A non-publishing demonstration is [`.github/workflows/publish-dry-run.yml`](../.github/workflows/publish-dry-run.yml). Running it with `workflow_dispatch` calls the reusable workflow with `dry-run: true`, which installs, builds, tests, checks the tarball, and runs `npm publish --dry-run` without writing to the registry.

### Tag format

Tags must follow the pattern `<component>-v<version>` where:
- `<component>` is one of: `sdk`, `cli`, `react`, `indexer`
- `<version>` is a valid semver version (e.g., `1.0.0`, `1.2.3-beta.1`)

Examples:
- `sdk-v1.0.0`
- `cli-v2.1.0`
- `react-v1.0.0-beta.1`
- `indexer-v0.5.0`

### Release process

1. Ensure all changes for the component are merged to `main`
2. Update the version in the component's `package.json`
3. Create and push the tag:
   ```bash
   git tag sdk-v1.0.0
   git push origin sdk-v1.0.0
   ```
4. The corresponding workflow will trigger automatically
5. Verify the publish succeeded in GitHub Actions and on npm

### Testing tag patterns locally

Run the tag validation script to verify a tag maps to exactly one component:

```bash
node scripts/validate-tag.js sdk-v1.0.0
# sdk-v1.0.0 -> sdk (version: 1.0.0, workflow: publish-sdk.yml)

node scripts/validate-tag.js cli-v2.0.0 react-v1.0.0-beta.1 indexer-v0.5.0
# each tag maps to exactly one component

node scripts/validate-tag.js v1.0.0
# ERROR: Tag "v1.0.0" does not match any component pattern
```

An unrelated tag such as `v1.0.0` or `release-v1.0.0` matches no workflow, so nothing is published.

## Legacy Changesets publishing (deprecated)

> **Note:** The following describes the previous Changesets-based workflow which is being phased out in favor of component-specific tag publishing.

`@bc-forge/sdk`, `@bc-forge/cli`, and `@bc-forge/react` previously published from
[`.github/workflows/release.yml`](../.github/workflows/release.yml) on a push to
`main`. Changesets opened the version PR, and the publish step built each
package, set `npm config set provenance true`, and ran `npx changeset publish`.

The workflow granted `id-token: write` so npm could verify the GitHub Actions OIDC
token. It did not pass `NODE_AUTH_TOKEN`. A long-lived npm token was not part of
the normal publish path.

### One-time npm trusted-publisher setup

Do this once per package (`@bc-forge/sdk`, `@bc-forge/cli`, `@bc-forge/react`, `@bc-forge/indexer`)
in the npm organization that owns the scope:

1. Sign in to [npmjs.com](https://www.npmjs.com) as an owner of the `bc-forge` organization.
2. Open the package, then **Settings → Trusted Publisher**.
3. Add a GitHub Actions publisher:
   - Organization or user: `BCPathway`
   - Repository: `bc-forge`
   - Workflow filename: `publish-sdk.yml` (or `publish-cli.yml`, `publish-react.yml`, `publish-indexer.yml`)
   - Environment name: leave empty unless a GitHub Environment is added to the release job later
4. Save. Repeat for the other packages.
5. Confirm **Access** is public for each package. The changesets config sets `"access": "public"`.
6. After the next release, open the package's **Versions** page and confirm the version shows a provenance attestation. The statement is also linked from the GitHub Actions run of the component publish workflow.

Provenance is requested by `npm config set provenance true` before `npm publish`. Pull requests do not publish.

## Fallback secret and rotation

Use a granular npm token only when trusted publishing is unavailable (for example, the publisher record has not been created yet).

1. On npm, create a **granular access token** that can publish only `@bc-forge/sdk`, `@bc-forge/cli`, `@bc-forge/react`, and `@bc-forge/indexer`. Do not create a classic token with access to every package you own.
2. Store it as the `NPM_TOKEN` Actions secret on `BCPathway/bc-forge`.
3. The reusable workflow accepts `NPM_TOKEN` as an optional secret for fallback publishing.
4. Rotate the secret after that publish, and after any exposure:
   - Revoke the token on npm (**Access Tokens → Revoke**).
   - Create a replacement granular token with the same package list.
   - Update the `NPM_TOKEN` repository secret. GitHub does not show the old value; replacing the secret is the rotation.
   - Delete the secret entirely once trusted publishing is confirmed on a release page.

Do not leave `NODE_AUTH_TOKEN` in the workflow after the fallback publish. A provenance publish that always sends a long-lived token is not trusted publishing.

## Verify a release

[`.github/workflows/publish-release-manifest.yml`](../.github/workflows/publish-release-manifest.yml) runs when a GitHub Release is published. It builds `@bc-forge/sdk`, `@bc-forge/cli`, and `@bc-forge/react`, packs each tarball, builds `bc_forge_token.wasm`, and builds the indexer image. It attaches `checksums.txt`, `manifest.json`, the three tarballs, and the token WASM to that release. `manifest.json` lists every one of those files with its component, version, filename, and SHA-256 checksum, plus the indexer image name and `containerimage.digest`.

Download `checksums.txt` and the artifacts into the same directory, then recompute the checksums.

Linux:

```bash
sha256sum -c checksums.txt
```

macOS:

```bash
shasum -a 256 -c checksums.txt
```

Windows PowerShell:

```powershell
Get-Content checksums.txt | ForEach-Object {
  $hash, $name = $_ -split '\s+', 2
  $actual = (Get-FileHash -Algorithm SHA256 -Path $name).Hash.ToLower()
  if ($actual -ne $hash) { throw "$name checksum mismatch" }
  Write-Output "$name OK"
}
```

A matching command prints `OK` for each file. A mismatch prints a checksum error and a non-zero exit status.

The indexer entry in `manifest.json` uses `digest` (`sha256:...`) rather than a filename. Compare that value to `containerimage.digest` in the "Build indexer image and record its digest" log of the release workflow. That digest is the image built for the release; it is not a GHCR pull digest, because this repository does not push the indexer image.