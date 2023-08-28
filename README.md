# pve-kernel-upstream

This is a fork of [Proxmox VE's official Linux kernel packaging tools](https://git.proxmox.com/git/pve-kernel.git). It has the changes and documentation needed to base builds on "upstream" (official) sources from [kernel.org](https://kernel.org/), instead of using Ubuntu's kernel source tree.

This might be useful if you need to mitigate security vulnerabilities that are patched upstream but haven't yet been backported to Ubuntu and/or Proxmox, and aren't easy to backport yourself.

## Important warnings

**You should only build your own kernel if you have a specific, good reason to do so!**

Unofficial kernel builds may miss Ubuntu patches that are important for features, stability, or security. They will be tested much less than an official Ubuntu or Proxmox build. You won't be able to get technical support for them.

This fork removes several patches that Proxmox backported, but are now included in *newer* upstream kernel versions. If you build a kernel version from before mid-August 2023, you will miss important security patches.

There is **no guarantee** that this fork will be actively maintained.

This fork disables backwards compatibility checks for ABI and firmware. You'll need to update all your kernel-related `.deb` packages together. Your built packages might not have their dependency metadata set correctly in order to force this to happen, like the official packages would.

`submodules/ubuntu-kernel` is now mis-named. This might cause confusion, but it seems cleaner than deleting and re-creating the submodule.

## Prerequisites

Make sure you have at least 35 GB of disk space available, and have these packages installed:

```bash
sudo apt-get install debhelper devscripts equivs git
```

## Clone this repository

```bash
git clone https://github.com/jprenken/pve-kernel-upstream
cd pve-kernel-upstream
```

## Update files for your target kernel version (optional)

This repository is probably ready to use for a specific kernel version (see [changelog](debian/changelog)), but that might not be the version you want. If it is, you can skip this whole section.

### Choose a version

Use a kernel *at least* as new as the minor version your Proxmox VE release uses. For example, Proxmox VE 8.0 expects at least Linux kernel 6.2.

See:
* [Proxmox VE Kernel](https://pve.proxmox.com/wiki/Proxmox_VE_Kernel)
* [Active kernel releases](https://kernel.org/category/releases.html)

As of this writing (August 2023), versions 6.2 and 6.3 are no longer maintained upstream; the stable kernel version series is 6.4. That makes the 6.4 series a good choice to use with Proxmox VE 8.0 because it's actively maintained, it's considered stable, and it's as close as possible to what Proxmox expects.

### Change the submodule repository (optional)

**Only if** you don't want to use the official stable/long-term kernel repository, change the submodule to use the repository you want. For example:

```bash
git submodule set-url -- submodules/ubuntu-kernel ${REPOSITORY_URL}
```

### Change the Ubuntu build scripts & configuration (optional)

Even though this fork uses the upstream kernel source, it still depends on some of Ubuntu's build scripts and configuration.

You may want to use newer or different versions than are in this repository. If so, replace the files in [`ubuntu/`](ubuntu/) with versions from an appropriate branch of Ubuntu's kernel source repository.

For example, here is the main branch for Ubuntu 23.04 (Lunar Lobster), which targets kernel version 6.1 as of this writing (August 2023):
* [https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/lunar/tree/](https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/lunar/tree/)

(This could be made into another submodule, but that would add complexity and consume much more storage space.)

### Update Makefile

At the top of [`Makefile`](Makefile), update the `KERNEL_MAJ`, `KERNEL_MIN`, and `KERNEL_PATCHLEVEL` variables to match your target kernel version.

If another kernel with the same minor version has already been packaged for Proxmox, also update `KREL`.

### Update the Debian packaging metadata

The build tools parse and depend on [`debian/changelog`](debian/changelog) for important metadata. Add a new entry, for example:

```bash
export NAME="Your Name"
export EMAIL="yourname@example.com"
debchange --newversion 6.4.12-1 --package proxmox-kernel-6.4 --no-auto-nmu
```

**If** you're switching to a different minor version than this repository currently uses, update the package name in [`debian/source/lintian-overrides`](debian/source/lintian-overrides) to avoid a pedantic build failure:

```bash
editor debian/source/lintian-overrides
```

## Install dependencies

Generate the Debian control file, then install the packages it says will be needed:

```bash
make debian.prepared
sudo mk-build-deps --install debian/control
```

## Download the kernel source tree

Although `make` will do this for you, it will try to use the recorded commit ID from git's internal state, which is probably wrong.

Download the tree and checkout your preferred [branch or tag](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git), for example:

```bash
git submodule update --init --remote submodules/ubuntu-kernel
git -C submodules/ubuntu-kernel checkout v6.4.12
```

## Build the kernel packages

```bash
make
```

## Troubleshoot build failures

You'll probably have several build failures. Many of these are easy to fix.

### Incompatible patches

If a patch fails to apply, inspect the patch file in [`patches/kernel/`](patches/kernel/) and figure out what's going on. Maybe:

* The patch, or a better version of it, has already landed in the upstream kernel.
* The patch is clearly not important to you, and safe to remove.
* Something has changed and you need to re-backport the patch yourself.

To remove a patch, just delete the file. Missing patch numbers are okay, as long as they stay in the correct order.

### Missing dependencies

Some build dependencies might not have been detected by `mk-build-deps`. Install them with `sudo apt-get install ${PACKAGE}`, and try again.

If the build process asks for `libpve-common-perl` or another package that's specific to Proxmox, you'll need to install it and its dependencies from a [Proxmox package repository](https://pve.proxmox.com/wiki/Package_Repositories). Because that may override or conflict with your system's normal packages, you might want to do this on a dedicated/temporary build host or container, instead of your workstation.

### ZFS test failure

If ZFS tests fail because `common.sh` doesn't exist, there's a simple workaround:

```bash
make --directory proxmox-kernel-*/modules/pkg-zfs/scripts
```

(This only happens sometimes. It may be related to a [known bug](https://github.com/openzfs/zfs/issues/14027) that's been [fixed](https://github.com/openzfs/zfs/pull/14051) in OpenZFS upstream.)

## Credits

These steps are based on [Fabian Mastenbroek](https://github.com/fabianishere)'s [pve-edge-kernel](https://github.com/fabianishere/pve-edge-kernel) documentation.