KERNEL SOURCE:
==============

We currently use the Ubuntu kernel sources, available from our mirror:

 https://git.proxmox.com/?p=mirror_ubuntu-kernels.git;a=summary

Ubuntu will maintain those kernels till:

 https://wiki.ubuntu.com/Kernel/Dev/ExtendedStable
 or
 https://pve.proxmox.com/pve-docs/chapter-pve-faq.html#faq-support-table

 whatever happens to be earlier.


Additional/Updated Modules:
---------------------------

- include native OpenZFS filesystem kernel modules for Linux

  * https://github.com/zfsonlinux/

  For licensing questions, see: http://open-zfs.org/wiki/Talk:FAQ


SUBMODULE
=========

We track the current upstream repository as submodule. Besides obvious
advantages over tracking binary tar archives this also has some implications.

For building the submodule directory gets copied into build/ and a few patches
get applied with the `patch` tool. From a git point-of-view, the copied
directory remains clean even with extra patches applied since it does not
contain a .git directory, but a reference to the (still pristine) submodule:

$ cat build/ubuntu-kernel/.git

If you mistakenly cloned the upstream repo as "normal" clone (not via the
submodule mechanics) this means that you have a real .git directory with its
independent objects and tracking info when copying for building, thus git
operates on the copied directory - and "sees" that it was dirtied by `patch`,
and thus the kernel buildsystem sees this too and will add a '+' to the version
as a result. This changes the output directories for modules and other build
artefacts and let's then the build fail on packaging.

So always ensure that you really checked it out as submodule, not as full
"normal" clone. You can also explicitly set the LOCALVERSION variable to
undefined with: `export LOCALVERSION= but that should only be done for test
builds.

RELATED PACKAGES:
=================

proxmox-ve
----------

top level meta package, depends on current default kernel series meta package.

git clone git://git.proxmox.com/git/proxmox-ve.git

proxmox-default-kernel
----------------------

Depends on default kernel and header meta package, e.g., proxmox-kernel-6.2 /
proxmox-headers-6.2.

git clone git://git.proxmox.com/git/pve-kernel-meta.git

proxmox-kernel-X.Y
------------------

Depends on the latest kernel (or header, in case of proxmox-headers-X.Y)
package within a certain series.

e.g., proxmox-kernel-6.2 depends on proxmox-kernel-6.2.16-6-pve

pve-firmware
------------

Contains the firmware for all released PVE kernels.

git clone git://git.proxmox.com/git/pve-firmware.git


NOTES:
======

ABI versions, package versions and package name:
------------------------------------------------

We follow debian's versioning w.r.t ABI changes:

https://kernel-team.pages.debian.net/kernel-handbook/ch-versions.html
https://wiki.debian.org/DebianKernelABIChanges

The debian/rules file has a target comparing the build kernel's ABI against the
version stored in the repository and indicates when an ABI bump is necessary.
An ABI bump within one upstream version consists of incrementing the KREL
variable in the Makefile, rebuilding the packages and running 'make abiupdate'
(the 'abiupdate' target in 'Makefile' contains the steps for consistently
updating the repository).

Watchdog blacklist
------------------

By default, all watchdog modules are black-listed because it is totally undefined
which device is actually used for /dev/watchdog.
We ship this list in /lib/modprobe.d/blacklist_proxmox-kernel-<VERSION>.conf
The user typically edit /etc/modules to enable a specific watchdog device.

Debug kernel and modules
------------------------

In order to build a -dbgsym package containing an unstripped copy of the kernel
image and modules, enable the 'pkg.proxmox-kernel.debug' build profile (e.g. by
exporting DEB_BUILD_PROFILES='pkg.proxmox-kernel.debug'). The resulting package can
be used together with 'crash'/'kdump-tools' to debug kernel crashes.

Note: the -dbgsym package is only valid for the proxmox-kernel packages produced by
the same build. A kernel/module from a different build will likely not match,
even if both builds are of the same kernel and package version.

Additional information
----------------------

We use the default configuration provided by Ubuntu, and apply
the following modifications:

NOTE: For the exact and current list see debian/rules (PVE_CONFIG_OPTS)

- enable INTEL_MEI_WDT=m (to allow disabling via patch)

- disable CONFIG_SND_PCM_OSS (enabled by default in Ubuntu, not needed)

- switch CONFIG_TRANSPARENT_HUGEPAGE to MADVISE from ALWAYS

- enable CONFIG_CEPH_FS=m (request from user)

- enable common CONFIG_BLK_DEV_XXX to avoid hardware detection
  problems (udev, update-initramfs have serious problems without that)

  	 CONFIG_BLK_DEV_SD=y
  	 CONFIG_BLK_DEV_SR=y
  	 CONFIG_BLK_DEV_DM=y

- compile NBD and RBD modules
	 CONFIG_BLK_DEV_NBD=m
	 CONFIG_BLK_DEV_RBD=m

- enable IBM JFS file system as module
  requested by users (bug #64)

- enable apple HFS and HFSPLUS as module
  requested by users

- enable CONFIG_BCACHE=m (requested by user)

- enable CONFIG_BRIDGE=y
  to avoid warnings on boot, e.g. that net.bridge.bridge-nf-call-iptables is an unknown key

- enable CONFIG_DEFAULT_SECURITY_APPARMOR
  We need this for lxc

- set CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y
  because if not set, it can give some dynamic memory or cpu frequencies 
  change, and vms can crash (mainly windows guest).
  see http://forum.proxmox.com/threads/18238-Windows-7-x64-VMs-crashing-randomly-during-process-termination?p=93273#post93273

- use 'deadline' as default scheduler
  This is the suggested setting for KVM. We also measure bad fsync performance with ext4 and cfq.

- disable CONFIG_INPUT_EVBUG
  Module evbug is not blacklisted on debian, so we simply disable it to avoid
  key-event logs (which is a big security problem)

- enable CONFIG_MODVERSIONS (needed for ABI tracking)

- switch default UNWINDER to FRAME_POINTER
  the recently introduced ORC_UNWINDER is not 100% stable yet, especially in combination with ZFS

- enable CONFIG_PAGE_TABLE_ISOLATION (Meltdown mitigation)
