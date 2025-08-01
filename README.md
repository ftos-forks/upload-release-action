# Upload files to a GitHub release [![GitHub Actions Workflow](https://github.com/svenstaro/upload-release-action/actions/workflows/e2e_test.yml/badge.svg)](https://github.com/svenstaro/upload-release-action/actions)

This action allows you to select which files to upload to the just-tagged release.
It runs on all operating systems types offered by GitHub.

## Input variables

You must provide:

- `file`: A local file to be uploaded as the asset.

### Optional Arguments


| **Argument**       | **Default**                     | **Description**                                                                                                                                                                   |
|--------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `repo_token`       | `github.token`                  | Defaults to `github.token`.                                                                                                                                                       |
| `tag`              | `github.ref`                    | The tag to upload into. If you want the current event's tag or branch name, use `${{ github.ref }}` (the `refs/tags/` and `refs/heads/` prefixes will be automatically stripped). |
| `asset_name`       | Filename                        | The name the file gets as an asset on a release. Use `$tag` to include the tag name. This is not used if `file_glob` is set to `true`.                                            |
| `file_glob`        | `false`                         | If set to true, the `file` argument can be a glob pattern (`asset_name` is ignored in this case).                                                                                 |
| `overwrite`        | `false`                         | If an asset with the same name already exists, overwrite it.                                                                                                                      |
| `check_duplicates` | `true`                          | Check for existing assets with the same name. Disabling removes this validity check, and allows reduced Github API usage when there are a large number of files.                  |
| `promote`          | `false`                         | If a prerelease already exists, promote it to a release.                                                                                                                          |
| `draft`            | `false`                         | Sets the release as a draft instead of publishing it, allowing you to make any edits needed before releasing.                                                                     |
| `release_id`       | ---                             | Used for searching for existing release, instead of tag. Must be used if uploading files to an existing draft release.                                                            |
| `prerelease`       | `false`                         | Mark the release as a pre-release.                                                                                                                                                |
| `make_latest`      | `true`                          | Mark the release as the latest release for the repository.                                                                                                                        |
| `release_name`     | Same as `tag`                   | Explicitly set a release name.                                                                                                                                                    |
| `target_commit`    | Default branch (usually `main`) | Sets the commit hash or branch for the tag to be based on.                                                                                                                        |
| `body`             | `""`                            | Content of the release text.                                                                                                                                                      |
| `repo_name`        | Current repository              | Specify the name of the GitHub repository in which the GitHub release will be created, edited, and deleted.                                                                       |

## Output variables

- `browser_download_url`: The publicly available URL of the asset.
- `release_id`: The numerical ID of the created or updated release.

## Usage

This usage assumes you want to build on tag creations only.
This is a common use case as you will want to upload release binaries for your tags.

Simple example:

```yaml
name: Publish

on:
  push:
    tags:
      - '*'

jobs:
  build:
    name: Publish binaries
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
    - name: Build
      run: cargo build --release
    - name: Upload binaries to release
      uses: svenstaro/upload-release-action@v2
      with:
        repo_token: ${{ secrets.GITHUB_TOKEN }}
        file: target/release/mything
        asset_name: mything
        tag: ${{ github.ref }}
        overwrite: true
        body: "This is my release text"
```

Complex example with more operating systems:

```yaml
name: Publish

on:
  push:
    tags:
      - '*'

jobs:
  publish:
    name: Publish for ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            artifact_name: mything
            asset_name: mything-linux-amd64
          - os: windows-latest
            artifact_name: mything.exe
            asset_name: mything-windows-amd64
          - os: macos-latest
            artifact_name: mything
            asset_name: mything-macos-amd64

    steps:
    - uses: actions/checkout@v4
    - name: Build
      run: cargo build --release --locked
    - name: Upload binaries to release
      uses: svenstaro/upload-release-action@v2
      with:
        repo_token: ${{ secrets.GITHUB_TOKEN }}
        file: target/release/${{ matrix.artifact_name }}
        asset_name: ${{ matrix.asset_name }}
        tag: ${{ github.ref }}
```

Example with `file_glob`:

```yaml
name: Publish
on:
  push:
    tags:
      - '*'

jobs:
  build:
    name: Publish binaries
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build
      run: cargo build --release
    - name: Upload binaries to release
      uses: svenstaro/upload-release-action@v2
      with:
        repo_token: ${{ secrets.GITHUB_TOKEN }}
        file: target/release/my*
        tag: ${{ github.ref }}
        overwrite: true
        file_glob: true
```

Example for creating a release in a foreign repository using `repo_name`:

```yaml
name: Publish

on:
  push:
    tags:
      - '*'

jobs:
  build:
    name: Publish binaries
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
    - name: Build
      run: cargo build --release
    - name: Upload binaries to release
      uses: svenstaro/upload-release-action@v2
      with:
        repo_name: owner/repository-name
        # A personal access token for the GitHub repository in which the release will be created and edited.
        # It is recommended to create the access token with the following scopes: `repo, user, admin:repo_hook`.
        repo_token: ${{ secrets.YOUR_PERSONAL_ACCESS_TOKEN }}
        file: target/release/mything
        asset_name: mything
        tag: ${{ github.ref }}
        overwrite: true
        body: "This is my release text"
```

**Example for feeding a file from repo to the `body` tag:**

This example covers following points:
* Reading a file present on the repo. For example, `release.md` which is placed in root directory of the repo.
* Modify & push the `release.md` file before triggering this action (create tag for this example) to dynamically change the body of the release.

```yaml
name: Publish

on:
  push:
    tags:
      - '*'

jobs:

  build:
    name: Publish binaries
    runs-on: ubuntu-latest
         
    steps:
      - uses: actions/checkout@v4

      # This step reads a file from repo and use it for body of the release
      # This works on any self-hosted runner OS
      - name: Read release.md and use it as a body of new release
        id: read_release
        shell: bash
        run: |
          r=$(cat path/to/release.md)                       # <--- Read release.md (Provide correct path as per your repo)
          r="${r//'%'/'%25'}"                               # Multiline escape sequences for %
          r="${r//$'\n'/'%0A'}"                             # Multiline escape sequences for '\n'
          r="${r//$'\r'/'%0D'}"                             # Multiline escape sequences for '\r'
          echo "RELEASE_BODY=$r" >> $GITHUB_OUTPUT          # <--- Set environment variable

      - name: Upload Binaries to Release
        uses: svenstaro/upload-release-action@v2
        with:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          tag: ${{ github.ref }}
          body: |
            ${{ steps.read_release.outputs.RELEASE_BODY }}  # <--- Use environment variables that was created earlier

```

### Permissions

This actions requires writes access to the release. If you are encountering "resource not accessible by integration" errors, you will need to add the `contents: write` permission to the token using [granular permissions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#permissions):

```yaml
permissions:
  contents: write
```

By default, these permissions are granted on `push` but not on `pr` - and you should be wary of adding them to workflows that run on pr, as they allow [wide access to changing the entire repo's contents](https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps?apiVersion=2022-11-28#repository-permissions-for-contents)

## Releasing

To release this Action:

- Bump version in `package.json`
- Create `CHANGELOG.md` entry
- `npm update`
- `npm run all`
- `git commit -am <version>`
- `git tag -sm <version> <version>`
- `git push --follow-tags`
- Go to https://github.com/svenstaro/upload-release-action/releases and publish the new version
