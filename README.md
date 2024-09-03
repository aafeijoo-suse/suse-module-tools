# suse-module-tools

This package contains a collection of tools and configuration files for
handling kernel modules and setting module parameters. The configuration files
represent a carefully engineered, recommended default configuration. In
certain cases, it may be necessary to modify or revert some of these settings.
It's ok to do so, but make sure you know what you're doing if you do.

Please don't edit any of the configuration files shipped in this package.
Instead, copy the files from `/lib/modprobe.d` to `/etc/modprobe.d`, preserving
the file name, and edit the copy under `/etc/modprobe.d`.
Likewise for `/lib/depmod.d` vs. `/etc/depmod.d` and `/usr/lib/modules-load.d` vs.
`/etc/modules-load.d`.

To completely mask the directives in a configuration file, it's recommended
to create a symlink to `/dev/null` with the same name as the file to be masked 
in the respective directory under `/etc`. E.g. to mask 
`/lib/modprobe.d/20-foo.conf`, run

    ln -s /dev/null /etc/modprobe.d/20-foo.conf


## Blacklisted file systems

In the Linux kernel, file system types are implemented as kernel modules. While
many of these file systems are well maintained, some of the older and less
frequently used ones are not. Others are actively maintained upstream but SUSE
kernel developers do not not actively review and include fixes for them in the
kernel sources released in openSUSE and SUSE Enterprise Linux. This poses a
security risk, because maliciously crafted file system images might open
security holes when mounted either automatically or by an inadvertent user.

Filesystems not actively maintained by SUSE kernel developers are therefore
**blacklisted** by default under openSUSE and SUSE Enterprise Linux. This means
that the on-demand loading of file system modules at mount time is disabled.
Blacklisting is accomplished by placing configuration files called
`60-blacklist_fs-$SOME_FS.conf` under `/lib/modprobe.d`. The current list of
blacklisted filesystems is:

    @FS_BLACKLIST@ # will be filled from spec file during package build

### CAVEAT

In the very unlikely case that one of the blacklisted file systems is necessary
for your system to boot, make sure you un-blacklist your file system before
rebooting.

### Un-blacklisting a file system

If a user tries to **mount(8)** a device with a blacklisted file system, the
mount command prints an error message like this:

    mount: /mnt/mx: unknown filesystem type 'minix' (hint: possibly blacklisted, see mount(8)).

(**mount(8)** can't distinguish between a file system for which no kernel
module exists at all, and a file system for which a module exists which
is blacklisted).

Users who need the blacklisted file systems and therefore want to override 
the blacklisting can load the blacklisted module directly using `modprobe
$SOME_FS` in a terminal. This will call a script that offers to "un-blacklist"
the module for future use.

    # modprobe minix
    unblacklist: *** NOTE: minix will be loaded even if you answer "n" below. ***
    unblacklist: minix is currently blacklisted, do you want to un-blacklist it (y/n)? y
    unblacklist: minix un-blacklisted by creating /etc/modprobe.d/60-blacklist_fs-minix.conf

If the user selects **y**, the module is un-blacklisted by creating a symlink
to `/dev/null` (see above). Future attempts to mount minix file systems will
work with no issue, even after reboot, because the kernel's auto-loading
mechanism works for this file system again. If the user selects **n**, the
module remains blacklisted. Regardless of the user's answer, the module will be
loaded for the time being; i.e. subsequent **mount** commands for devices with
this file system will succeed until the module is unloaded or the system is
rebooted.

For security reasons, it's recommended that you only un-blacklist file system
modules that you know you'll use on a regular basis, and just enable them
temporarily otherwise.


## Weak modules

This package contains the script `weak-modules2` which is necessary to make
3rd party kernel modules installed for one kernel available to
KABI-compatible kernels. SUSE ensures KABI compatibility over the life
time of a service pack in SUSE Enterprise Linux. See the
[SUSE SolidDriver Program](https://drivers.suse.com/doc/SolidDriver/) for
details.

### Capturing log output from weak_modules2

Use the following environment variables:

 * WM2_VERBOSE: value from 0 (default, no logging) - 3 (tracing).
   the -v/--verbose option increases log level by one.
 * WM2_DEBUG: 0 (default) or 1. Enables verbose output of certain
   commands called by weak-modules2. Equivalent to --debug.
 * WM2_LOGFILE: redirect the output to the given file.

## Kernel scriptlet files

The scripts in kernel-scriptlets directory are used internally by kernel
packages.

### Capturing log output from kernel scripts

 * KERNEL_PACKAGE_SCRIPT_DEBUG when non-empty enables some extra output to kernel log.

## Kernel-specific sysctl settings

This package installs the file `50-kernel-uname_r.conf` which makes sure
that sysctl settings which are recommended for the currently running kernel
are applied by **systemd-sysctl.service** at boot time. These settings are
shipped in the file `/boot/sysctl.conf-$(uname -r)`, which is part of the
kernel package.


# Boot Loader Specification (BLS) and EFI System Partition (ESP)

There are scripts generating boot entries (via perl-Bootloader), new
initrds (via dracut or mkosi-initrd) and updating the kernel module dependency
lists (via depmod).  If we are in a system using the boot entries defined in
the bootloader specification (BLS), then we need to take special
considerations.

The tool `sdbootutil` is the one responsible of synchronizing the
content in the rootfs with the ESP (kernel, bootloader, shim, boot
entries, initrd, etc), so certain actions should be delegated to it.

To complicate the situation further, transactional systems (like
MicroOS) cannot access the ESP from inside the transaction as that
would break the atomicity of the update operation.  It is the
`snapper` plugin provided by `sdbootutil` that will call the scripts
in the correct moment (when setting the new default snapshot).

This imposes that `weak-modules2`, `regenerate-initrd-posttrans`, the
`kernel-scriptlets` and the snapper plugin now need to work in
coordination, reordering certain actions depending on the kind of
system.  The following table summarizes those interactions:


| Model            | Operation       | Element    | Done                           |
|------------------|-----------------|------------|--------------------------------|
| Traditional      | Kernel          | depmod     | wm2 (rpm-script/post[un])      |
|                  |                 | initrd     | wm2 (rpm-script/post[un])      |
|                  |                 | boot entry | rpm-script/post[un]            |
|                  |                 |            |                                |
|                  | KMP             | depmod     | wm2 (inkmp-script/post[un])    |
|                  |                 | initrd     | wm2 (inkmp-script/post[un])    |
|                  |                 |            |                                |
|                  | dracut /        | initrd     | regenerate-initrd-posttrans[2] |
|                  | mkosi-initrd[1] |            |                                |
|------------------|-----------------|------------|--------------------------------|
| MicroOS[3]       | Kernel          | depmod     | wm2 (rpm-script/post[un])      |
|                  |                 | initrd     | wm2 (rpm-script/post[un])      |
|                  |                 | boot entry | rpm-script/post[un]            |
|                  |                 |            |                                |
|                  | KMP             | depmod     | wm2 (inkmp-script/post[un])    |
|                  |                 | initrd     | wm2 (inkmp-script/post[un])    |
|                  |                 |            |                                |
|                  | dracut          | initrd     | regenerate-initrd-posttrans    |
|------------------|-----------------|------------|--------------------------------|
| Tumbleweed + BLS | Kernel          | depmod     | wm2 (rpm-script/post[un])[4]   |
|                  |                 | initrd     | wm2 (rpm-script/post[un])      |
|                  |                 | boot entry | rpm-script/post[un]            |
|                  |                 |            |                                |
|                  | KMP             | depmod     | wm2 (inkmp-script/post[un])    |
|                  |                 | initrd     | wm2 (inkmp-script/post[un])    |
|                  |                 |            |                                |
|                  | dracut          | initrd     | regenerate-initrd-posttrans    |
|------------------|-----------------|------------|--------------------------------|
| MicroOS + BLS    | Kernel          | depmod     | wm2 (rpm-script/post[un])      |
|                  |                 | initrd     | snapper plugin[5]              |
|                  |                 | boot entry | snapper plugin                 |
|                  |                 |            |                                |
|                  | KMP             | depmod     | snapper plugin[6]              |
|                  |                 | initrd     | wm2 (rpm-script/post[un])      |
|                  |                 |            |                                |
|                  | dracut          | initrd     | snapper plugin                 |
|------------------|-----------------|------------|--------------------------------|
| Tumbleweed + BLS | Kernel          | depmod     | wm2 (rpm-script/post[un])[7]   |
| (no btrfs)       |                 | initrd     | wm2 (rpm-script/post[un])      |
|                  |                 | boot entry | rpm-script/post[un]            |
|                  |                 |            |                                |
|                  | KMP             | depmod     | wm2 (inkmp-script/post[un])    |
|                  |                 | initrd     | wm2 (inkmp-script/post[un])    |
|                  |                 |            |                                |
|                  | dracut          | initrd     | regenerate-initrd-posttrans    |
|------------------|-----------------|------------|--------------------------------|

Notes:

[1] The `mkosi-initrd` integration is in its initial phase and is intended for
    traditional systems only.

[2] Triggered by the `%regenerate_initrd_post[trans]` macros

[3] In MicroOS (or any system that use transactional-update) the
    kernel in /boot is inside the transaction, so gets discarded if
    the snapshot is dropped.

[4] Could be done in the snapper plugin, but it is done in
    weak-modules2 as in the traditional case, by calling `sdbootutil
    --no-reuse-initrd`, which also creates the boot entry.  The initrd
    name is selected from the current default boot entry

[5] When adding or removing a kernel, the `sdbootutil
    set_default_snapshot` will regenerate boot entries for all the
    remaining kernels in the snapshot.  This will synchronize also the
    initrds (but can leave old initrds in the ESP).  Also, wm2 will
    create a mark in `/run/regenerate-initrd`.

[6] A direct call to `regenerate-initrd-posttrans` inside the
    transaction will drop the call and keep the
    `/run/regenerate-initrd` directory.  A second call (from the
    snapper plugin) will complete it.

[7] `sdbootutil` partially understand BLS systems without snapshots.
