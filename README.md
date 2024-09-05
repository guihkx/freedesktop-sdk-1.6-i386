## About

This repository contains instructions on how to easily install the i386 build of the [Freedesktop 1.6 SDK](https://github.com/flatpak/freedesktop-sdk-images).

The 1.6 SDK is old and unmaintained, so it's not used anywhere on Flathub except to package [NVIDIA drivers for Flatpak](https://github.com/flathub/org.freedesktop.Platform.GL.nvidia).

And even though we can still download and install the x86_64 build of 1.6 SDK directly from Flathub, the i386 build is not public anymore and it's only used privately by Flathub's build bot.

So, the ultimate purpose of this repository is to allow Flatpak users with NVIDIA GPUs to easily create a complete driver package locally, without having to wait on Flathub's long build/publish times.

## Installing

> [!NOTE]
> These are all command-line instructions.

**1\.** Download the latest `.tar.zst` archive of the OSTree repository for the i386 Freedesktop 1.6 SDK:

```bash
curl -LO https://github.com/guihkx/freedesktop-sdk-1.6-i386/releases/download/1.6-2024-09-03/ostree-repo-freedesktop-sdk-1.6-i386.tar.zst
```

**2\.** Decompress the `.tar.zst` archive:

```bash
tar --warning=no-timestamp -xf ostree-repo-freedesktop-sdk-1.6-i386.tar.zst
```

**3\.** You can now remove the downloaded `.tar.zst` archive:

```bash
rm ostree-repo-freedesktop-sdk-1.6-i386.tar.zst
```

**4\.** Create a local Flatpak remote named `sdk-1.6-i368`, pointing to the `sdk-repo/` directory we just extracted:

```bash
flatpak --user remote-add --no-gpg-verify sdk-1.6-i386 sdk-repo/
```

**5\.** You can now install the i386 1.6 Platform/SDK:

```bash
flatpak --user install --no-related sdk-1.6-i386 org.freedesktop.Platform/i386/1.6
flatpak --user install --no-related sdk-1.6-i386 org.freedesktop.Sdk/i386/1.6
```

**6\.** You can now perform a general clean up:

```bash
# Disable the sdk-1.6-i368 Flatpak remote (we're not outright deleting it because that causes 'flatpak update' to display an annoying warning)
flatpak --user remote-modify --disable sdk-1.6-i386
# Delete the 'sdk-repo' repository
rm -rf sdk-repo/
```

Please note that the commands above **will not** also uninstall the i386 Freedesktop Platform/SDK.

If you really want to do that, run these commands:

```bash
flatpak --user uninstall org.freedesktop.Platform/i386/1.6
flatpak --user uninstall org.freedesktop.Sdk/i386/1.6
flatpak --user remote-delete --force sdk-1.6-i386
```

## Advanced

This section explains how you can create a local build of the i386 1.6 SDK.

This is a long process that requires lots of disk space, internet bandwidth, computer resources, and patience. I recommended you just download and use the pre-built OSTree repository of the i386 1.6 SDK, as explained in the [Installing](#installing) section above.

Disclaimer aside, you'll only need these two programs installed so we can begin:

- Git
- Podman (tested only with v4.9.3)

**1\.** Clone this repository:

```
git clone --recurse-submodules https://github.com/guihkx/freedesktop-sdk-1.6-i386.git
cd freedesktop-sdk-1.6-i386/
```

**2\.** We first have to build the i386 1.6 Freedesktop BasePlatform/BaseSdk runtimes, in order to later build the i386 Freedesktop 1.6 SDK itself:

```bash
# Create a base container with pre-installed tools for building the BasePlatform/BaseSdk.
podman build -t freedesktop-sdk-base -f Dockerfile.base .
# The next two steps are needed so that Yocto doesn't yell at us for running it as "root".
mkdir -p freedesktop-sdk-base/build/i386/conf
touch freedesktop-sdk-base/build/i386/conf/sanity.conf
# Invoke the build process. NOTE: This might take hours.
podman run --rm -v $(pwd):/src -it freedesktop-sdk-base make -C freedesktop-sdk-base ARCH=i386 REPO=../sdk-base-repo
```

**3\.** We can now finally build the i386 Freedesktop 1.6 SDK:

```bash
# Create a base container with pre-installed tools for building the Platform/Sdk.
podman build -t freedesktop-sdk -f Dockerfile.main .
# Invoke the build process. NOTE: This might take hours.
# We must invoke 'podman run' with extended capabilities here, otherwise bubblewrap (used by flatpak-builder) fails with these errors:
# - bwrap: Creating new namespace failed: Operation not permitted (fixed by the sys_admin capability)
# - bwrap: loopback: Failed RTM_NEWADDR: No child processes (fixed by the net_admin capability)
podman run --cap-add net_admin,sys_admin --rm -v $(pwd):/src -t freedesktop-sdk make -C freedesktop-sdk-images ARCH=i386 FB_ARGS='--disable-rofiles-fuse --force-clean' REPO=../sdk-repo
# Lastly, we must update the OSTree repository after building, because after the 'flatpak build-commit-from' call in 'freedesktop-images/Makefile:84',
# all previous refs (sdk, platform, docs, etc.) get hidden (for some reason), and without this we'd be unable to install them with 'flatpak install'.
podman run --rm -v $(pwd)/sdk-repo:/sdk-repo -t freedesktop-sdk flatpak build-update-repo --no-summary-index /sdk-repo
```

That's it! You should find your OSTree repository of the i386 1.6 SDK under the `sdk-repo` directory.

You can now follow the [Installing](#installing) section above to learn how to install the SDK from the repository.
