# libostree

    - Operating systems and distributions using OSTree
    - Distribution build tools
    - Projects linking to libostree
    - Language bindings
    - Building
    - Contributing
    - Licensing

This project is now known as “libostree”, though it is still appropriate to use the previous name: “OSTree” (or “ostree”). The focus is on projects which use libostree’s shared library, rather than users directly invoking the command line tools (except for build systems). However, in most of the rest of the documentation, we will use the term “OSTree”, since it’s slightly shorter, and changing all documentation at once is impractical. We expect to transition to the new name over time.

As implied above, libostree is both a shared library and suite of command line tools that combines a “git-like” model for committing and downloading bootable filesystem trees, along with a layer for deploying them and managing the bootloader configuration.

The core OSTree model is like git in that it checksums individual files and has a content-addressed-object store. It’s unlike git in that it “checks out” the files via hardlinks, and they thus need to be immutable to prevent corruption. Therefore, another way to think of OSTree is that it’s just a more polished version of Linux VServer hardlinks.

**Features:**

    Transactional upgrades and rollback for the system
    Replicating content incrementally over HTTP via GPG signatures and “pinned TLS” support
    Support for parallel installing more than just 2 bootable roots
    Binary history on the server side (and client)
    Introspectable shared library API for build and deployment systems
    Flexible support for multiple branches and repositories, supporting projects like flatpak which use libostree for applications, rather than hosts.

## Operating systems and distributions using OSTree

Endless OS uses libostree for their host system as well as flatpak. See their eos-updater and deb-ostree-builder projects.

Fedora derivatives use rpm-ostree (noted below); there are 3 variants using OSTree:

    Fedora CoreOS
    Fedora Silverblue
    Fedora IoT

Red Hat Enterprise Linux CoreOS is a derivative of Fedora CoreOS, used in OpenShift 4. The machine-config-operator manages upgrades. RHEL CoreOS is also the successor to RHEL Atomic Host, which uses rpm-ostree as well.

GNOME Continuous is where OSTree was born - as a high performance continuous delivery/testing system for GNOME.

Liri OS has the option to install their distribution using ostree.

TorizonCore uses libostree and Aktualizr as the base for OTA updates from compatible platforms, including Torizon OTA.

## Distribution build tools

meta-updater is a layer available for OpenEmbedded systems.

QtOTA is Qt’s over-the-air update framework which uses libostree.

The BuildStream build and integration tool supports importing and exporting from libostree repos.

Fedora coreos-assembler is the build tool used to generate Fedora CoreOS derivatives.

## Projects linking to libostree

rpm-ostree is used by the Fedora-derived operating systems listed above. It is a full hybrid image/package system. By default it uses libostree to atomically replicate a base OS (all dependency resolution is done on the server), but it supports “package layering”, where additional RPMs can be layered on top of the base. This brings a “best of both worlds”” model for image and package systems.

eos-updater is a daemon that implements updates on EndlessOS.

flatpak uses libostree for desktop application containers. Unlike most of the other systems here, flatpak does not use the “libostree host system” aspects (e.g. bootloader management), just the “git-like hardlink dedup”. For example, flatpak supports a per-user OSTree repository.

## Language bindings

libostree is accessible via GObject Introspection; any language which has implemented the GI binding model should work. For example, Both pygobject and gjs are known to work and further are actually used in libostree’s test suite today.

Some bindings take the approach of using GI as a lower level and write higher level manual bindings on top; this is more common for statically compiled languages. Here’s a list of such bindings:

    ostree-go
    ostree-rs

## Building

Releases are available as GPG signed git tags, and most recent versions support extended validation using git-evtag.

However, in order to build from a git clone, you must update the submodules. If you’re packaging OSTree and want a tarball, I recommend using a “recursive git archive” script. There are several available online; this code in OSTree is an example.

Once you have a git clone or recursive archive, building is the same as almost every autotools project:

git submodule update --init
env NOCONFIGURE=1 ./autogen.sh
./configure --prefix=...
make
make install DESTDIR=/path/to/dest

## Contributing

See Contributing.

## Licensing

The licensing for the code of libostree can be canonically found in the individual files; and the overall status in the COPYING file in the source. Currently, that’s LGPLv2+. This also covers the man pages and API docs.

The license for the manual documentation in the doc/ directory is: SPDX-License-Identifier: (CC-BY-SA-3.0 OR GFDL-1.3-or-later) This is intended to allow use by Wikipedia and other projects.

In general, files should have a SPDX-License-Identifier and that is canonical.

Copyright © Red Hat, Inc. and others.

Edit this page on GitHub
