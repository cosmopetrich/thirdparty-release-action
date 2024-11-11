# Usage 

This package was built with [vcpkg](https://github.com/microsoft/vcpkg) for use in Satisfactory modding. Specifically, it:

- Comes with the directory structure that UE expects.
- Includes libraries for both Linux and Windows.
- Has support for static linking.
- Was built with the appropriate C compiler version to avoid mismatches with Unreal Engine.

## Adding the library to your mod

The headers and library files are not checked in to git since they might include large binaries that git does not handle well,
and public repositories can quickly hit the monthly usage limits that Github imposes on git-lfs.
Instead, they are distributed using Github Releases. You can grab the newest release from
[the releases page](https://github.com/cosmopetrich/ue-thirdparty-release-action/releases) and
[set it up as a third-party library](https://docs.ficsit.app/satisfactory-modding/latest/Development/Cpp/thirdparty.html)
manually if desired. For some smaller libraries that update infrequently it may be perfectly okay to check the binaries in to git.

Alternatively, read on for information on grabbing the release files in a semi-automated way.
The commands below assume that you're using the "Git Bash" command-line included in Git for Windows.

1. Add this repository as a [git submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules).

   ```bash
   cd SatisfactoryModLoader/Mods/MyMod
   git submodule add https://github.com/cosmopetrich/ue-thirdparty-release-action Source/foolib
   ```

2. Update your mod's Build.cs file as normal.

   ```cs
   PrivateDependencyModuleNames.AddRange(new string[] {
     // [...]
     "foolib",
     // [...]
   });
   ```

3. Add a note to your `CONTRIBUTING.md` or equivilant so that contributors know what to do.

   ```markdown
   When updating the repository, be sure to also update the submodules.

       git pull --recurse-submodules

   You can also set git up to do this automatically when you run a regular `git pull`.

       git config submodule.recurse true
   ```

3. Retrieve the files.

   ```bash
   Source/foolib/update-files
   ```

If the command exists successfully then you'll have `include` and `lib` folders containing the library and can develop your mod as usual.

To update to a newer version of this package you can simply update the submodule in git and run the update-files script again.

1. Update the submodule.

   ```bash
   git -C Source/foolib pull
   git commit -m "Update foolib" Source/foolib .gitmodules
   ```

2. Re-run the `update-files script.


   ```bash
   Source/foolib/update-files
   ```

It's safe to run `update-files` whenever you want to. If nothing needs to change then it will exit without doing anything. 
