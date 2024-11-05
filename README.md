# ue-thirdparty-release-action

Arranges files into a structure suitable for use as a "ThirdParty" Unreal Engine dependency.

This is intended primarily for personal use. If you've happened across it then it is _probably_ not what you're looking for.

It is designed for use with the [cosmopetrich/vcpkg-manifest-install-action](https://github.com/cosmopetrich/vcpkg-manifest-install-action)
action and probably won't be much good without, unless care is taken to arrange the inputs in the same manner as that action does.

Possible future improvements include:

- Have the action abort if there's an open PR against the target branch.
  Will probably need to munge the tests to make that work, though. This is trivial to implement, but it will need to be optiona or the tests will need to be adpated for it.
- Or possibly have it update the existing PR/release, if one exists.
- Support for packaging release versions of libraries (i.e. without debug libs).
  Not currently needed and unlikely to be needed since this was made for modding.

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

## Repository and workflow requirements

### Actions user permissions

Under the repository's Actions/General settings page, ensure that the following are enabled.

- Workflow permissions
  - "Read and write permissions"
  - "Allow GitHub Actions to create and approve pull requests "

### PR/Issue label

The action will assign a label to the pull requests that it creates to help distinguish them.
This feature currently is not optional (though the label can be changed with the `pr-label` input),
and the lable should be created ahead of time.

1.  From the main repository page, hit either "Issues' or "Pull Requests".
2.  Select the "Labels" button to the right of the search bar, near the "New {Issue,Pull Request}" button.
3.  Add a new label with the chosen name. The action's default is `automated-release`.

### Workflow permissions

Within the workflow itself, ensure that the permissions required to commit, create releases, and create PRs are granted.
Additionally, the workflow should have at least the `contents: write` permission.

```yaml
permissions:
  contents: write
  pull-requests: write
```

### Workflow triggers

It is recommended that the list of paths which trigger the build be as restrictive as possible to prevent
builds from triggering based on the actions bot's own PRs and commits.

```yaml
name: Build
on:
  workflow_dispatch:
  pull_request:
    branches:
      - master
    paths:
      - .vcpkg/**
  push:
    branches:
      - master
    paths:
      - .vcpkg/**
```

### Branch protection

While not an explicit requirement, it is recommended that any important branches (main/master, etc) be
protected with branch rules. While the action is not desgined to interact with these branches, adding rules
can help protect against unexpected failure scenarios.

Enabling the "Require a pull request before merging" option with the "Require review from Code Owners" and
"Dismiss stale pull request approvals when new commits are pushed" sub-options should be enough to prevent the
action from doing anything unfortunate since Github Actions cannot bypass these restrictions.

Additionally, disabling non-rebase merges in the repository's main settings page may help ensure that
merges are fast-forward only if possible, reducing the likelihood that releases must be manually updated.

## Details

At a high level this action does the following.

- Parse the artifacts directory to determine information needed to create a Github Release.
- Shuffle the include/debug folders from the artifacts directory into a structure that UE expects.
  - Headers are drawn from an arbitrary platform, Win64 by default.
  - Also merge the tools folders of each artifact, if that folder is present.
  - The debug folder is renamed to lib. This means that, currently, `-release` vcpkg tuplets are not supported.
- Add or update files to the repository that allow the update-files helper script to run.
- Generate Build.cs file and add that.
- Add some other static files.
  - update-files helper script
  - gitattributes and gitignore
- Create a new branch, commit, push, update the release to point at the new commit.
- Open a PR to merge the branch.

### Release naming

The release will be named according to the format `[{prefix}]{upstream-version}[-{revision}]' where:

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

### Artifact naming

The artifact will be named according to the format `{package}-{release}.zip` where:

- `{package}` is the name provided to the `package-name` input.
- `{release}` is the release **without** the prefix, i.e. `{upstream-version}[-{revision}]`.

If `package-name` was provided with some specific casing then it will be preserved in the artifact name.
For example, `package-name: uWebSockets` will produce an artifact like "uWebSockets-1.2.3.zip" even if
the package name in vcpkg is "uwebsockets".

### Expected format of artifacts directory

The `artifacts-directory` input is expected to contain the following:

- `Linux/` - Results of package installation on Linux.
- `Win64/` - Results of package installation on Windows.
- `{Linux,Win64}/vpckg-build-info.json` - Information on the vcpkg environment used to install the packages.
- `{Linux,Win64}/vcpkg-installed-packages.txt` - Plain text list of package names and versions present.

### Use of a PR vs directly committing

Early versions of this action committed directly to master (or equivilant).
However, Github Actions cannot currently bypass branch protection rules (
community discussion [#13836](https://github.com/orgs/community/discussions/13836)
and [#25305](https://github.com/orgs/community/discussions/25305))
which would necessitate leaving protections for human users disabled.

The workaround for this is to have the action create a pull request.
While it would be very possible to have it merge its own PR,
currently it will not do so since leaving the PR open provides
a handy way for a human to check that it isn't doing anything stupid.

One downside tot his is that since pull requests cannot force fast-forward-only
(see [community discussion #4618](https://github.com/orgs/community/discussions/4618))
the user who merges the PR will need to update the release manually if any commits have gone through to master since it was created
(even if the commit only does something trivial like update the README).
