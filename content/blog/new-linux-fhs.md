---
title: The New Linux Filesystem Hierarchy Standard
draft: true
---

# The New Linux Filesystem Hierarchy Standard

The UNIX Filesystem Hierarchy Standard is "a set of requirements and guidelines
for where files should be located in UNIX-like operating systems"
([FHS 3.0][fhs-3.0]).
Developed in 1994, this spec was originally a [descriptive][linguistic-prescription]
account of UNIX systems that was meant to prescribe a standard onto developers,
administrators, and maintainers.

In the decades since, Linux and other UNIX systems have changed a lot
and obsoleted many of the directories in the specification.
By looking at how distributions today use these hierarchies,
I hope to provide a more useful description of the filesystem hierarchy for
developers and administrators.

A few things before we start:

1. This focuses on Linux, but many of the decisions are relevant to other
   UNIX-like systems.
   I'll mention where FreeBSD differs from Linux, as it is a BSD
   system I'm somewhat familiar with.

2. The examples and explanations cite `systemd` and FreeDesktop resources.
   Distributions that use other init systems such as OpenRC and runit
   follow the same hierarchy, but for the examples I have to make a choice.

3. Some of the suggestions here are glimpses into the future, and are about
   specifying how distributions would *like* their packages to work,
   rather than how they work in practice.
   I'm trying describe on how distributions prefer the hierarchy to look like.

## The Modern Hierarchy

### /proc

The mount point for the [`procfs`][man-procfs] filesystem.
Files under this directory are used to query information about running processes.
Ex. `/proc/<pid>/cmdline` contains the command line invocation used to
launch the process with id `<pid>`.
Note that this is not part of the UNIX FHS, and the structure is different
between different UNIXes
(Ex. Linux: `/proc/self/exe`, FreeBSD: `/proc/curproc/file`).

### /sys

The mount point for the [`sysfs`][man-sysfs] filesystem.
Files under this directory are views into the Linux kernel's internal components.
Linux kernel modules are encouraged to use this directory to communicate
with userspace. <!-- TODO: cite -->

#### History

This directory is Linux-specific, and is not present on BSD.
Historically `procfs` was used for all kinds of kernel-userspace communication
on Linux, which overcomplicated the `procfs` code.
To keep the `procfs` filesystem code minimal, Linux 2.6 introduced `sysfs`.
BSD's `procfs` was never extended in the same way, and similar functionality
is obtained through syscalls.

### /dev

Directory that stores devices.
On Linux this is most likely mounted as a `devtmpfs` filesystem,
and on FreeBSD this is mounted as the `devfs` filesystem.
The special pseudo-devices `/dev/null`, `/dev/zero`, `/dev/random`,
and `/dev/tty` should [always be available][systemd-exec-PrivateDevices],
so developers should be able to rely on their existence.
The other files and directories are used to reference devices such as drives,
partitions, USB video cameras, etc.

### /usr -- Installed system; sharable; possibly read-only

This directory should now be viewed as "the Operating System".
A Linux distribution is a service that manages the `/usr/` directory.
This means that as a developer, you should not be modifying anything
in this hierarchy.

One of Leonard Poettering's goals is to allow a Linux system to boot the
first time entirely off a single `/usr` partition.
During setup, `systemd-boot` will create + resize three partitions --
one for the current `/usr`, one for a new `/usr` for system upgrades,
and a partition for all other data.
This setup is great because it enables `dm-verity` checks; because `/usr` is
immutable, it can be signed and integrity can be preserved through boot.
Keeping all distribution files under one directory also allows for atomic
upgrades, where the new OS version is installed in a second partition,
and the `/usr` directory is replaced.

If your system embraces this philosophy,
an administrator cannot touch this directory at all.
And if you system has a mutable `/usr`, an administrator should still refrain
from modifying files under `/usr` whenever possible.
The one carve-out for local administration is `/usr/local`,
meant for packages managed outside the distribution.

### /etc -- Configuration data

A Linux system will ship with default configurations, but system settings
can be overridden with files in `/etc`.
Files should not be stored directly under `/etc`, packages should create
directories.

### /var -- Persistent data



### /run -- Volatile data
### /home -- User directories

## Rarely used Directories

### /opt -- Linux's "C:\Program Files"

Both `/opt` and `/usr/local` were historically used to store packages 
that were not part of a distribution, but were instead built by a
system administrator.
`/usr/local` has a fairly strict hierarchy, and packages installed
to `/usr/local` should have the same installed file structure as `/usr`.
`/opt` is instead organized into per-package directories,
and thus can accommodate unusual file structures.

Ex. if you have a package that looks like this:

```
.
|-- LICENSE
|-- bin
|   `-- myprog
|-- config
|   `-- config.conf
|-- doc
|   `-- usage.pdf
`-- scripts
    `-- setup.sh
```

Then you don't want to install this under `/usr/local`.
One option is to install this directory under `/usr/local/lib/<package>`,
but another is to install this under `/opt/<package>`.
The administrator can then symlink user-facing binaries to `/opt/bin`
to make it easier for users to add the package to `$PATH`.

One argument for this directory structure is for easy versioning,
since `/opt/mypackage-1.2` and `/opt/mypackage-2.0` can coexist.
Depending on the use-case, tools like [GNU stow][stow]
can be used to version packages in `/usr/local` too.

As a developer, this directory should not be explicitly mentioned.
There are issues with `/opt` directories on some distributions,
for example Fedora Silverblue has had issues with Google Chrome because 
of its insistence on using `/opt` directory paths.
As mentioned previously, distributions with a hermetic `/usr` will have
trouble including your package if it stores data under `/opt`.

### /mnt -- Temporary directory for mounts

This directory is meant to be a temporary mount point that the system
administrator can use at their discretion.
This directory is often used as a location to store long-lasting
mounts, though this usage is not part of the FHS.

## Deprecated Directories

### /bin /sbin /lib /lib64

These directories are now (almost) universally symlinked to `/usr/<path>`
to maintain compatibility,
but new code should use `/usr/<path>` when possible.
Keeping the directories separate is called "split-usr", and the
change has sometimes called "the /usr merge".
`systemd`'s reasoning is [here][usr-merge-reason].

`systemd` has not supported "split-usr" setups since 2022,
and `OpenRC` 

#### History

TODO: Explain

### /usr/sbin

The *system binaries* directory is sometimes still separate from `/usr/bin`,
but many Linux systems
(Arch [since 2013][arch-bin-sbin],
Fedora [since 2025][fedora-bin-sbin],
)
`systemd` added this as a taint in 2024, so assume more `systemd`-based
Linux distributions to follow suit.

## The FreeDesktop Base Directory Specification

The FreeDesktop Base Directory Specification is an attempt to standardize
file locations for user programs.

### $HOME/.local/bin

Analogue of `/usr/bin`, user binaries should be installed here.
It should be on the user's `$PATH`.

### $XDG\_CONFIG\_HOME -- ~/.config

Analogue of `/etc`, this directory should contain a directory for each package.
Configuration data should be stored here.

### $XDG\_DATA\_HOME -- ~/.local/share

Analogue of `/usr/share`, this directory should store read-mostly data
for a package.

### $XDG\_STATE\_HOME -- ~/.local/state

Analogue of `/var/lib`, this directory should store files that a package
modifies during runtime.

### $XDG\_CACHE\_HOME -- ~/.cache

Analogue of `/var/cache`, this directory should store files that a package
writes which can be safely deleted without issue.

### $XDG\_RUNTIME\_DIR -- /run/user/$ID

Analogue of `/run`, this directory is created by PAM on user login.
This meant to be the proper location for data related to the current login.
For example, USB drives can be auto-mounted under `/run/user/$ID/media`.

### $XDG\_CONFIG\_DIRS, $XDG\_DATA\_DIRS

These environment variables are search paths, and tell packages where to
look for configuration / data files after looking in
`$XDG_CONFIG_HOME`/`$XDG_DATA_HOME`.
By default and when unset, the config file location is `/etc/xdg` and
the data dirs are `/usr/local/share:/usr/share`.
Packages can install default configuration or shared data to these directories,
but should allow users to override the defaults.

[fhs-3.0]: https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html
[linguistic-prescription]: https://en.wikipedia.org/wiki/Linguistic_prescription
[man-sysfs]: https://man7.org/linux/man-pages/man5/sysfs.5.html
[man-procfs]: https://man7.org/linux/man-pages/man5/proc.5.html
[systemd-exec-PrivateDevices]: https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#PrivateDevices=
[arch-bin-sbin]: https://archlinux.org/news/binaries-move-to-usrbin-requiring-update-intervention/
[fedora-bin-sbin]: https://fedoraproject.org/wiki/Changes/Unify_bin_and_sbin
[usr-merge-reason]: https://systemd.io/THE_CASE_FOR_THE_USR_MERGE/
[stow]: https://www.gnu.org/software/stow/
