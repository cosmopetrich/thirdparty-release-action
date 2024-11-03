# ue-thirdparty-release-action

Arranges files into a structure suitable for use as a "ThirdParty" Unreal Engine dependency.

This is intended primarily for personal use. If you randomly stumbled across it then it is *probably* not what you're looking for.

This is designed for use with [cosmopetrich/vcpkg-manifest-install-action](https://github.com/cosmopetrich/vcpkg-manifest-install-action)
and probably won't be much good without it unless care is taken to arrange the inputs in the same manner as that action does.

## Usage

At a minimum it needs to know the location of the artifacts, the name of the package, and where the repository is checked out.

```yaml
- name: release
  uses: cosmopetrich/ue-thirdparty-release-action@v1
  with:
    artifacts-directory: artifacts
    package-name: uwebsockets
    repo-directory: repo
```

See below for details on what the action will do with these options.

## Details

At a high level this action does the following.

 - Parse the artifacts directory to determine information needed to create a Github Release.
 - Zip the artifacts directory, excluding only the vcpkg info files, and upload it to the release.
 - Generate SHA256 checksums of the files in the release and the release artifact itself and place them in the repository.
 - Write the URL of the release artifact to the repository.
 - Commit and push the modified files.
 - Update the release to point at the new commit.

## Expected format of artifacts directory

The `artifacts-directory` input is expected to contain the following:

- `Linux/` - Results of package installation on Linux.
- `Win64/` - Results of package installation on Windows.
- `*/vpckg-build-info.json` - Information on the vcpkg environment used to install the packages.
- `*/vcpkg-installed-packages.txt` - Plain text list of package names and versions present.

Any other contents of the Linux and Win64 directories are opaque to this action.
Everything except for the vcpkg info files will be added to the release artifact.
