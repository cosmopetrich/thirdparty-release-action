# ue-thirdparty-release-action

Arranges files into a structure suitable for use as a "ThirdParty" Unreal Engine dependency.

This is intended primarily for personal use. If you randomly stumbled across it then it is _probably_ not what you're looking for.

This is designed for use with [cosmopetrich/vcpkg-manifest-install-action](https://github.com/cosmopetrich/vcpkg-manifest-install-action)
and probably won't be much good without it unless care is taken to arrange the inputs in the same manner as that action does.

## Usage


At a minimum it needs to know the location of the artifacts, the name of the package, and where the repository is checked out.

```yaml
# Trigger only on pushes to master
- name: release
  if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/master' }}
  uses: cosmopetrich/ue-thirdparty-release-action@v1
  with:
    artifacts-directory: artifacts
    package-name: uwebsockets
    repo-directory: repo
```

Note that the action will fail if the repository at `repo-directory` is in a "detached head" state,
which is the default when using the checkout action on pull requests. However, this action almost
certainly shouldn't be run against PRs.

## Workflow requirements

Since this action creates a release and commits to the repo, the "read and write" option for the Actions token must be enabled in the repository's settings
Additionally, the workflow should have at least the `contents: write` permission.

```yaml
permissions:
  contents: write
  pull-requests: write
```

Additionally, it is recommended that the list of paths which trigger the build be as restrictive as possible to prevent
builds from triggering based on the bot's own actions.

```yaml
name: Build
on:
  workflow_dispatch:
  pull_request:
    branches:
      - master
    paths:
      - vcpkg.json
  push:
    branches:
      - master
    paths:
      - vcpkg.json
```

Github Actions cannot currently bypass branch protection rules (
community discussion [#13836](https://github.com/orgs/community/discussions/13836)
and [#25305](https://github.com/orgs/community/discussions/25305).
If the target branch is 

Pull requests also cannot force fast-forward-only, see [community discussion #4618](https://github.com/orgs/community/discussions/4618).

## Details

At a high level this action does the following.

- Parse the artifacts directory to determine information needed to create a Github Release.
- Shuffle the include/lib folders from the artifacts directory into a structure that UE expects.
  - Headers are drawn from an arbitrary platform, Win64 by default.
  - Also merge the tools folders of each artifact, if that folder is present.
- Add or update files to the repository that allow the update-files helper script to run.
- Generate Build.cs file and add that.
- Add some other static files.
  - update-files helper script
  - gitattributes and gitignore
- Commit, push, update the release to point at the new commit.

The resulting commit will be left as a draft so that it can be manually editted prior to release.

## Release naming

The release will be named according to the format `[{prefix}]{upstream-version}[-{revision}] where:

- `{prefix}` is an arbitrary string that defaults to "v".
  It can be altered using one of the action's inputs.
- `{upstream-version}` is the version of the package specified in the `package-name` input which was built by vcpkg.
- `{revision}` is added if a release named `[{prefix}]{upstream-version}` already exists,
  which could occur if vcpkg updates its port without targetting a new upstream release,
  or if configuration in this repository (such as the Build.cs file) is updated.
  It begins at "-1" and increments for each such release.

Note that the resulting version numbers are not valid [semver](https://semver.org/),
even if they may resemble it depending on the version format used by the upstream package.
They were styled after the [Debian Pacakge Policy](https://www.debian.org/doc/debian-policy/ch-controlfields.html#s-f-version).

## Artifact naming

The artifact will be named according to the format `{package}-{release}.zip` where:

- `{package}` is the name provided to the `package-name` input.
- `{release}` is the release **without** the prefix, i.e. `{upstream-version}[-{revision}]`.

If `package-name` was provided with some specific casing then it will be preserved in the artifact name.
For example, `package-name: uWebSockets` will produce an artifact like "uWebSockets-1.2.3.zip" even if
the package name in vcpkg is "uwebsockets".

## Expected format of artifacts directory

The `artifacts-directory` input is expected to contain the following:

- `Linux/` - Results of package installation on Linux.
- `Win64/` - Results of package installation on Windows.
- `{Linux,Win64}/vpckg-build-info.json` - Information on the vcpkg environment used to install the packages.
- `{Linux,Win64}/vcpkg-installed-packages.txt` - Plain text list of package names and versions present.

## Side-effects

Although the action takes care to operate outside the root of the Actions workspace where possible,
it will still have the following side-effects on the rest of the build.

- Some git config fields will be set for the provided `repo-directory`, including `core.safecrlf` and user name/email.
