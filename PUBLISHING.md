# Publishing to npm

This fork is published to npm as [`@jaivalsaija/react-rough-notation`](https://www.npmjs.com/package/@jaivalsaija/react-rough-notation) by GitHub Actions.

There are a few ways to set up publishing. This repo uses the **recommended** one. The others are documented so you can switch, or use them in other projects.

| Option | Triggered by | Credentials | Status |
| --- | --- | --- | --- |
| [A. Publish when the version changes](#a-publish-when-the-version-changes-recommended) | Push to `main` with a new version | Trusted publishing (no token) | **Used in this repo. Recommended.** |
| [B. Publish on version tags](#b-publish-on-version-tags) | Pushing a `vX.Y.Z` tag | Trusted publishing (no token) | Widely used, fully supported. |
| [C. Staged publishing (manual approval)](#c-staged-publishing-manual-approval) | Either of the above | Trusted publishing or a stage-only token | Supported. Adds a manual approval step. |
| [D. npm token secret](#d-npm-token-secret-legacy) | Either of the above | `NPM_TOKEN` repository secret | Legacy. Direct publishing with tokens is removed in **January 2027**. |

## How trusted publishing works

Options A, B and C use [trusted publishing](https://docs.npmjs.com/trusted-publishers). Instead of storing an npm token in GitHub, the workflow asks GitHub for a short-lived identity token (OIDC) that says which repository and workflow file is running. npm checks it against the trusted publisher configured for the package and only accepts the publish if they match. There's no secret to leak, rotate or let expire. npm also attaches a [provenance](https://docs.npmjs.com/generating-provenance-statements) record to every version, which links it to the commit and run that built it.

Requirements:

- npm CLI 11.5.1 or newer and Node 22.14.0 or newer. The workflows use Node 24, which includes a new enough npm.
- `repository.url` in `package.json` must point at the GitHub repository that publishes.
- The workflow needs `permissions: id-token: write`.

## One-time setup

### 1. `package.json`

```json
{
  "name": "@jaivalsaija/react-rough-notation",
  "files": ["dist"],
  "scripts": {
    "build": "rollup -c",
    "test": "npm run build",
    "prepublishOnly": "npm run build"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/Jaival/react-rough-notation.git"
  },
  "publishConfig": { "access": "public" }
}
```

- Scoped packages (`@you/name`) are private by default. `publishConfig.access: "public"` makes them public.
- `prepublishOnly` builds `dist/` right before every publish. Don't name a script `publish`: npm runs a script with that name _after_ publishing, so `npm publish` would run twice.
- `dist/` is gitignored. It's build output, and committing it leaves the working tree dirty after every build, which makes `npm version` fail with `Git working directory not clean`. npm still includes it because it's listed in `files`.

### 2. Publish the first version by hand

The package's settings page, where trusted publishing is configured, only exists after the first publish.

```sh
npm login
npm publish
```

### 3. Configure the trusted publisher

On npmjs.com, open the package, go to **Settings → Trusted Publisher → GitHub Actions**, and enter:

| Field | Value |
| --- | --- |
| Organization or user | `Jaival` |
| Repository | `react-rough-notation` |
| Workflow filename | `npm-publish.yml` (the filename only, not the path) |
| Environment | leave empty |
| Allowed actions | allow `npm publish` (options A and B), or staging only (option C) |

### 4. Lock down publishing

In the package's **Settings → Publishing access**, choose **Require two-factor authentication and disallow tokens**. Releases can then only come from the trusted workflow, or from you interactively with 2FA. Delete any npm tokens you no longer need.

Skip this step if you use [option D](#d-npm-token-secret-legacy), because it blocks tokens.

## A. Publish when the version changes (recommended)

This is the workflow in [`.github/workflows/npm-publish.yml`](.github/workflows/npm-publish.yml). It runs on every push to `main` and:

1. Reads `name` and `version` from `package.json`.
2. Skips the rest if that version is already on npm, so ordinary pushes don't fail.
3. Otherwise installs, builds, publishes, and pushes a `vX.Y.Z` tag for the release.

```yaml
name: NPM Package Publish

on:
  push:
    branches:
      - main
  workflow_dispatch:

concurrency:
  group: npm-publish
  cancel-in-progress: false

permissions:
  id-token: write # Required for OIDC
  contents: write # Required to push the version tag

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: '24'
          registry-url: 'https://registry.npmjs.org'
          package-manager-cache: false

      - name: Check whether this version is already published
        id: version
        run: |
          NAME=$(node -p "require('./package.json').name")
          VERSION=$(node -p "require('./package.json').version")
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          if npm view "$NAME@$VERSION" version > /dev/null 2>&1; then
            echo "$NAME@$VERSION is already on npm, skipping publish."
            echo "publish=false" >> "$GITHUB_OUTPUT"
          else
            echo "Publishing $NAME@$VERSION."
            echo "publish=true" >> "$GITHUB_OUTPUT"
          fi

      - if: steps.version.outputs.publish == 'true'
        run: npm ci
      - if: steps.version.outputs.publish == 'true'
        run: npm test
      - if: steps.version.outputs.publish == 'true'
        run: npm publish

      - name: Tag the release
        if: steps.version.outputs.publish == 'true'
        run: |
          TAG="v${{ steps.version.outputs.version }}"
          if git ls-remote --exit-code --tags origin "refs/tags/$TAG" > /dev/null; then
            echo "$TAG already exists."
          else
            git tag "$TAG"
            git push origin "$TAG"
          fi
```

**To release:**

```sh
npm version patch   # or minor / major: updates package.json, commits, tags locally
git push
```

The workflow publishes and creates the tag. `git push --follow-tags` also works; the workflow sees the tag already exists and moves on.

**Why it's recommended:** a release is just a version bump on `main`, so the published package always matches `main`. Pushes without a version bump are no-ops instead of failures, and you can't forget to push the tag.

## B. Publish on version tags

This repo used this setup before switching to option A, and it's still a very common pattern. Publishing happens only when you push a tag, so commits on `main` and releases are fully separate. The trusted publisher setup is the same as option A.

```yaml
name: NPM Package Publish

on:
  push:
    tags:
      - 'v*'

permissions:
  id-token: write # Required for OIDC
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: '24'
          registry-url: 'https://registry.npmjs.org'
          package-manager-cache: false
      - run: npm ci
      - run: npm test
      - run: npm publish
```

**To release:**

```sh
npm version patch
git push --follow-tags   # the tag push starts the workflow
```

**Watch out for:**

- Forgetting `--follow-tags`: the version bump lands on `main` but nothing is published.
- Pushing a tag whose version is already on npm fails with `E403`.
- Use this instead of A if you want to merge version bumps without releasing them immediately.

## C. Staged publishing (manual approval)

With [staged publishing](https://docs.npmjs.com/cli/commands/npm-stage), CI uploads the version but it doesn't go live until a maintainer approves it with 2FA. Use this if you want to review every release, for example in a package with several maintainers or automated dependency bumps.

1. In option A or B, replace `npm publish` with:

   ```yaml
   - run: npm stage publish
   ```

2. On the trusted publisher, allow staging only (not `npm publish`). Alternatively, use a **Read and write (stage only)** token as described in option D.

3. After the workflow runs, approve the release from your machine:

   ```sh
   npm stage list @jaivalsaija/react-rough-notation
   npm stage view <stage-id>              # optional: inspect it first
   npm stage approve <stage-id> --otp <code>
   # or: npm stage reject <stage-id>
   ```

For pre-release versions such as `1.2.0-beta.1`, pass a tag: `npm stage publish --tag beta`. The tag can't be changed after staging; reject and re-stage instead.

## D. npm token secret (legacy)

Before trusted publishing, the standard approach was to create an npm access token, store it as a GitHub secret, and pass it to `npm publish`. Many projects still do this, and it's the fallback for CI providers that don't support trusted publishing.

> [!WARNING]
> npm is removing the ability to **publish directly** with access tokens in **January 2027**. Tokens with **Bypass 2FA** are already restricted from account-level actions. After the cutoff, token-based publishing only works with **stage-only** tokens and `npm stage publish` (option C). Prefer trusted publishing for new setups.

**1. Create a granular access token** on npmjs.com (**Access Tokens → Generate New Token → Granular Access Token**):

| Setting | Direct publish (until January 2027) | Staged publish (long-term) |
| --- | --- | --- |
| Bypass two-factor authentication | Checked | Unchecked |
| Packages and scopes → Permissions | Read and write (publish and stage) | Read and write (stage only) |
| Select packages | Only `@jaivalsaija/react-rough-notation` | Only `@jaivalsaija/react-rough-notation` |
| Organizations | No access | No access |
| Expiration | As short as you're willing to rotate | As short as you're willing to rotate |

**2. Add it to GitHub:** repository **Settings → Secrets and variables → Actions → New repository secret**, named `NPM_TOKEN`. Or from the terminal:

```sh
gh secret set NPM_TOKEN -R Jaival/react-rough-notation
```

**3. Use it in the publish step** of option A or B. `actions/setup-node` writes an `.npmrc` that reads `NODE_AUTH_TOKEN`:

```yaml
- run: npm publish # or: npm stage publish, with a stage-only token
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

`permissions: id-token: write` isn't needed for this option. Package **Publishing access** must allow tokens: **Require two-factor authentication or a granular access token with bypass 2FA enabled**.

**Downsides:** the token is a long-lived secret that can leak, it expires and has to be rotated, publishes have no provenance unless you add `--provenance` (which needs `id-token: write` anyway), and direct publishing stops working in January 2027.

## Troubleshooting

| Error | Cause and fix |
| --- | --- |
| `E404 Not Found - PUT https://registry.npmjs.org/...` | No trusted publisher is configured, or the user, repository or workflow filename doesn't match exactly. Also check `permissions: id-token: write`. With a token: the token is missing, expired, or doesn't cover this package. |
| `E403 ... cannot publish over the previously published versions` | That version is already on npm. Bump it with `npm version patch`. |
| `E_STAGE_REQUIRED` | The credentials only allow staging. Use `npm stage publish` (option C), or allow `npm publish` on the trusted publisher. |
| `EOTP` | You're publishing interactively with 2FA enabled. Add `--otp <code>`. |
| `npm error Git working directory not clean` | `npm version` needs a clean tree. Commit or stash your changes first. If the changes are in `dist/`, make sure it's gitignored. |
| Provenance or repository mismatch | `repository.url` in `package.json` doesn't match the repository running the workflow. |
| The workflow succeeded but `npm view` doesn't show the new version | The registry can take a few minutes to update. Retry with `npm view <name> versions --prefer-online`. |
