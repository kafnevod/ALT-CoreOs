# Related Projects

    Combining dpkg/rpm + (BTRFS/LVM)
    ChromiumOS updater
    Ubuntu Image Based Updates
    Clear Linux Software update
    casync
    Mender.io
    OLPC update
    NixOS / Nix
    Solaris IPS
    Google servers (custom rsync-like approach, live updates)
    Conary
    bmap
    Git
    Conda
    rpm-ostree
    GNOME Continuous
    Docker
    Docker-related: Balena
    Torizon Platform
        TorizonCore
        TorizonCore Builder
        Torizon OTA
            Licensing for this document:

OSTree во многих отношениях очень быстро эволюционирует. 
Он основан на концепциях и идеях, представленных во многих различных проектах, таких как Systemd Stateless, Systemd Bootloader Spec, Chromium Autoupdate, Fedora / Red Hat Stateless Project, Linux VServer и многие другие.

Как упоминалось выше, OSTree также сильно зависит от дизайна диспетчера пакетов. 
Эта раздел не является исчерпывающим списком таких проектов, но мы постараемся поддерживать его в актуальном состоянии.

Вообще говоря, проекты в этой области делятся на две категори_
- инструмент для создания моментальных снимков систем на стороне клиента (dpkg / rpm + BTRFS / LVM);
- инструмент для создания репозитория на сервере и репликации их клиенту (ChromiumOS, Clear Linux). 

OSTree достаточно гибок, чтобы делать и то, и другое.

Обратите внимание, что этот раздел документации почти полностью посвящен модели «дерево для хоста»; 
Проект Flatpak использует libostree для хранения данных приложения, в отличие от модели управления хост-системой. 

## Combining dpkg/rpm + (BTRFS/LVM)

In this approach, one uses a block/filesystem snapshot tool underneath the system package manager.

The oVirt Node imgbased tool is an example of this approach, as are a few others below.

Regarding BTRFS in particular - the OSTree author believes that Linux storage is a wide world, and while BTRFS is quite good, it is not everywhere now, nor will it be in the near future. There are other recently developed filesystems like f2fs, and Red Hat Enterprise Linux still defaults to XFS.

Using a snapshot tool underneath a package manager does help significantly. In the rest of this text, we will use “BTRFS” as a mostly generic tool for filesystem snapshots.

The obvious thing to do is layer BTRFS under dpkg/rpm, and have a separate subvolume for /home so rollbacks don’t lose your data. See e.g. Fedora BTRFS Rollback Feature.

More generally, if you want to use BTRFS to roll back changes made by dpkg/rpm, you have to carefully set up the partition layout so that the files laid out by dpkg/rpm are installed in a subvolume to snapshot.

This problem in many ways is addressed by the changes OSTree forces, such as putting all local state in /var (e.g. /usr/local -> /var/usrlocal). Then one can BTRFS snapshot /usr. This gets pretty far, except handling /etc is messy. This is something OSTree does well.

In general, if one really tries to flesh out the BTRFS approach, a nontrivial middle layer of code between dpkg/rpm and BTRFS (or deep awareness of BTRFS in dpkg/rpm itself) will be required. A good example of this is the snapper.io project.

The OSTree author believes that having total freedom at the block storage layer is better for general purpose operating systems. For example, the ability to choose dm-crypt per deployment is quite useful; not every site wants to pay the performance penalty. One can choose LVM or not, etc.

Where applicable, OSTree does take advantage of copy-on-write/reflink features offered by the kernel for /etc. It uses the now generic ioctl(FICLONE) and copy_file_range().

Another major distinction between the default OSTree usage and package managers is whether updates are “online” or “offline” by default. The default OSTree design writes updates into a new root, leaving the running system unchanged. This means preparing updates is completely non-disruptive and safe - if the system runs out of disk space in the middle, it’s easy to recover. However, there is work in the rpm-ostree project to support online updates as well.

OSTree supports using “bare-user” repositories, which do not require root to use. Using a filesystem-level layer without root is more difficult and would likely require a setuid helper or privileged service.

Finally, see the next portion around ChromiumOS for why a hybrid but integrated package/image system improves on this.

## ChromiumOS updater

Many people who look at OSTree are most interested in using it as an updater for embedded or fixed-purpose systems, similar to use cases from the ChromiumOS updater.

The ChromiumOS approach uses two partitions that are swapped via the bootloader. It has a very network-efficient update protocol, using a custom binary delta scheme between filesystem snapshots.

This model even allows for switching filesystem types in an update.

A major downside of this approach is that the OS size is doubled on disk always. In contrast, OSTree uses plain Unix hardlinks, which means it essentially only requires disk space proportional to the changed files, plus some small fixed overhead.

This means with OSTree, one can easily have more than two trees (deployments). Another example is that the system OSTree repository could also be used for application containers.

Finally, the author of OSTree believes that what one really wants for many cases is image replication with the ability to layer on some additional components (e.g. packages) - a hybrid model. This is what rpm-ostree is aiming to support.

## Ubuntu Image Based Updates

See https://wiki.ubuntu.com/ImageBasedUpgrades. Very architecturally similar to ChromeOS, although more interesting is discussion for supporting package installation on top, similar to rpm-ostree package layering.

## Clear Linux Software update

The Clear Linux Software update system is not very well documented. This mailing list post has some reverse-engineered design documentation.

Like OSTree static deltas, it also uses bsdiff for network efficiency.

More information will be filled in here over time. The OSTree author believes that at the moment, the “CL updater” is not truly atomic in the sense that because it applies updates live, there is a window where the OS root may be inconsistent.

## casync

The systemd casync project is relatively new. Currently, it is more of a storage library, and doesn’t support higher level logic for things like GPG signatures, versioning information, etc. This is mostly the OstreeRepo layer. Moving up to the OstreeSysroot level - things like managing the bootloader configuration, and most importantly implementing correct merging for /etc are missing. casync also is unaware of SELinux.

OSTree is really today a shared library, and has been for quite some time. This has made it easy to build higher level projects such as rpm-ostree which has quite a bit more, such as a DBus API and other projects consume that, such as Cockpit.

A major issue with casync today is that it doesn’t support garbage collection on the server side. OSTree’s GC works symmetrically on the server and client side.

Broadly speaking, casync is a twist on the dual partition approach, and shares the general purpose disadvantages of those.

## Mender.io

Mender.io is another implementation of the dual partition approach.

## OLPC update

OSTree is basically a generalization of olpc-update, except using plain HTTP instead of rsync. OSTree has the notion of separate trees that one can track independently or parallel install, while still sharing storage via the hardlinked repository, whereas olpc-update uses version numbers for a single OS.

OSTree has built-in plain old HTTP replication which can be served from a static webserver, whereas olpc-update uses rsync (more server load, but more efficient on the network side). The OSTree solution to improving network bandwidth consumption is via static deltas.

See this comment for a comparison.

## NixOS / Nix

See NixOS. It was a very influential project for OSTree. NixOS and OSTree both support the idea of independent “roots” that are bootable.

In NixOS, files in a package are accessed by a path depending on the checksums of package inputs (build dependencies) - see Nix store. However, OSTree uses a commit/deploy model - it isn’t tied to any particular directory layout, and you can put whatever data you want inside an OSTree, for example the standard FHS layout. A both positive and negative of the Nix model is that a change in the build dependencies (e.g. being built with a newer gcc), requires a cascading rebuild of everything. It’s good because it makes it easy to do massive system-wide changes such as gcc upgrades, and allows installing multiple versions of packages at once. However, a security update to e.g. glibc forces a rebuild of everything from scratch, and so Nix is not practical at scale. OSTree supports using a build system that just rebuilds individual components (packages) as they change, without forcing a rebuild of their dependencies.

Nix automatically detects runtime package dependencies by scanning content for hashes. OSTree only supports only system-level images, and doesn’t do dependency management. Nix can store arbitrary files, using nix-store –add, but, more commonly, paths are added as the result of running a derivation file generated using the Nix language. OSTree is build-system agnostic; filesystem trees are committed using a simple C API, and this is the only way to commit files.

OSTree automatically shares the storage of identical data using hard links into a content-addressed store. Nix can deduplicate using hard links as well, using the auto-optimise-store option, but this is not on by default, and Nix does not guarantee that all of its files are in the content-addressed store. OSTree provides a git-like command line interface for browsing the content-addressed store, while Nix does not have this functionality.

Nix used to use the immutable bit to prevent modifications to /nix/store, but now it uses a read-only bind mount. The bind mount can be privately remounted, allowing per-process privileged write access. OSTree uses the immutable bit on the root of the deployment, and mounts /usr as read-only.

NixOS supports switching OS images on-the-fly, by maintaining both booted-system and current-system roots. It is not clear how well this approach works. OSTree currently requries a reboot to switch images.

Finally, NixOS supports installing user-specific packages from trusted repositories without requiring root, using a trusted daemon. Flatpak, based on OSTree, similarly has a policykit-based system helper that allows you to authenticate via polkit to install into the system repository.

## Solaris IPS

See Solaris IPS. Broadly, this is a similar design as to a combination of BTRFS+RPM/deb. There is a bootloader management system which combines with the snapshots. It’s relatively well thought through - however, it is a client-side system assembly. If one wants to image servers and replicate reliably, that’d be a different system.

## Google servers (custom rsync-like approach, live updates)

This paper talks about how Google was (at least at one point) managing updates for the host systems for some servers: Live Upgrading Thousands of Servers from an Ancient Red Hat Distribution to 10 Year Newer Debian Based One (USENIX LISA 2013)

## Conary

See Conary Updates and Rollbacks. If rpm/dpkg are like CVS, Conary is closer to Subversion. It’s not bad, but e.g. its rollback model is rather ad-hoc and not atomic. It also is a fully client side system and doesn’t have an image-like replication with deltas.

# bmap
See bmap. A tool for optimized copying of disk images. Intended for offline use, so not directly comparable.

## Git

Although OSTree has been called “Git for Binaries”, and the two share the idea of a hashed content store, the implementation details are quite different. OSTree supports extended attributes and uses SHA256 instead of Git’s SHA1. It “checks out” files via hardlinks, rather than copying, and thus requires the checkout to be immutable. At the moment, OSTree commits may have at most one parent, as opposed to Git which allows an arbitrary number. Git uses a smart-delta protocol for updates, while OSTree uses 1 HTTP request per changed file, or can generate static deltas.

## Conda

Conda is an “OS-agnostic, system-level binary package manager and ecosystem”; although most well-known for its accompanying Python distribution anaconda, its scope has been expanding quickly. The package format is very similar to well-known ones such as RPM. However, unlike typical RPMs, the packages are built to be relocatable. Also, the package manager runs natively on Windows. Conda’s main advantage is its ability to install collections of packages into “environments” by unpacking them all to the same directory. Conda reduces duplication across environments using hardlinks, similar to OSTree’s sharing between deployments (although Conda uses package / file path instead of file hash). Overall, it is quite similar to rpm-ostree in functionality and scope.

## rpm-ostree

This builds on top of ostree to support building RPMs into OSTree images, and even composing RPMs on-the-fly using an overlay filesystem. It is being developed by Fedora, Red Hat, and CentOS as part of Project Atomic.

## GNOME Continuous

This is a service that incrementally rebuilds and tests GNOME on every commit. The need to make and distribute snapshots for this system was the original inspiration for ostree.

## Docker

It makes sense to compare OSTree and Docker as far as wire formats go. OSTree is not itself a container tool, but can be used as a transport/storage format for container tools.

Docker has (at the time of this writing) two format versions (v1 and v2). v1 is deprecated, so we’ll look at format version 2.

A Docker image is a series of layers, and a layer is essentially JSON metadata plus a tarball. The tarballs capture changes between layers, including handling deleting files in higher layers.

Because the payload format is just tar, Docker hence captures (numeric) uid/gid and xattrs.

This “layering” model is an interesting and powerful part of Docker, allowing different images to reference a shared base. OSTree doesn’t implement this natively, but it’s not difficult to implement in higher level tools. For example in flatpak, there’s a concept of a SDK and runtime, and it would make a lot of sense for the SDK to depend on the runtime, to avoid clients downloading data twice (even if it’s deduplicated on disk).

That gets to an advantage of OSTree over Docker; OSTree checksums individual files (not tarballs), and uses this for deduplication. Docker (natively) only shares storage via layering.

The biggest feature OSTree has over Docker though is support for (static) deltas, and even without pre-configured static deltas, the archive format has “natural” deltas. Particularly for a “base operating system”, one really wants on-wire deltas. It’d likely be possible to extend Docker with this concept.

A core challenge both share is around metadata (particularly signing) and search/discovery (the ostree summary file doesn’t scale very well).

One major issue Docker has is that it checksums compressed data, and furthermore the tar format is flexible, with multiple ways to represent data, making it hard to impossible to reassemble and verify from on-disk state. The tarsum effort was intended to address this, but it was not adopted in the end for v2.

## Docker-related: Balena

The Balena project forks Docker and aims to even use Docker/OCI format for the root filesystem, and adds wire deltas using librsync. See also discussion on libostree-list.

## Torizon Platform

Torizon is an open-source software platform that simplifies the development and maintenance of embedded Linux software. It is designed to be used out-of-the-box on devices requiring high reliability, allowing you to focus on your application and not on building and maintaining the operating system.

### TorizonCore

The platform OS - TorizonCore - is a minimal OS with a Docker runtime and libostree + Aktualizr. The main goal of this system is to allow application developers to use containers, while the maintainers of TorizonCore focus on the base system updates.

### TorizonCore Builder

Since the TorizonCore OS is meant as a binary distribution, OS customization is made easier with TorizonCore Builder, as the tool abstracts the handling of OSTree concepts from the final users.

### Torizon OTA

Torizon OTA is a hosted OTA update system that provides OS updates to TorizonCore using OSTree and Aktualizr.
