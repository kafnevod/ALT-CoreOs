# libostree

    - Operating systems and distributions using OSTree
    - Distribution build tools
    - Projects linking to libostree
    - Language bindings
    - Building
    - Contributing
    - Licensing

Этот проект теперь известен как «libostree», хотя по-прежнему уместно использовать предыдущее название: «OSTree» (или «ostree»). Основное внимание уделяется проектам, которые используют общую библиотеку libostree, а не пользователям, напрямую вызывающим инструменты командной строки (за исключением систем сборки). Однако в большей части остальной документации мы будем использовать термин «OSTree», поскольку он немного короче, и изменять сразу всю документацию нецелесообразно. Мы ожидаем перехода на новое имя со временем.

Как упоминалось выше, libostree - это как совместно используемая библиотека, так и набор инструментов командной строки, которые объединяют «git-подобную» модель для фиксации и загрузки деревьев загрузочных файловых систем, а также уровень их развертывания и управления конфигурацией загрузчика.

Базовая модель OSTree похожа на git тем, что в ней проверяются суммы отдельных файлов и есть хранилище объектов с адресацией к содержимому. В отличие от git, он «проверяет» файлы с помощью жестких ссылок, и поэтому они должны быть неизменными, чтобы предотвратить повреждение. Таким образом  OSTree можно представить как  более совершенную версию жестких ссылок (hard links) Linux VServer. 

**Особенности:**

- Транзакционные обновления и откат для операционной системы
- Инкрементальная репликация контента через HTTP с помощью подписей GPG и поддержки «прикрепленного (pinned) TLS»
- Поддержка параллельной установки двух и более загрузочных корневых файловых систем
- Бинарная история на стороне сервера (и клиента) 
- Общий API для систем сборки и развертывания 
- Гибкая поддержка нескольких веток и репозиториев


## Операционные системы и дистрибутивы, использующие OSTree 

[Endless OS](https://endlessos.com/) использует libostree для своей хост-системы, а также flatpak. См. [eхos-updater](https://github.com/endlessm/eos-updater)
и [deb-ostree-builder](https://github.com/dbnicholson/deb-ostree-builder). 

Три подпроекта Fedora используют rpmostree: 
- [Fedora CoreOS](https://getfedora.org/en/coreos/)
- [Fedora Silverblue](https://silverblue.fedoraproject.org/)
- [Fedora IoT](https://getfedora.org/iot/)

Red Hat Enterprise Linux CoreOS - это производная от Fedora CoreOS, используемая в [OpenShift 4](https://www.openshift.com/try). 
[machine-config-operator](https://github.com/openshift/machine-config-operator/blob/master/docs/OSUpgrades.md) управляет обновлениями. RHEL CoreOS также является преемником RHEL At

omic Host, который также использует rpm-ostree. 

[GNOME Continuous](https://wiki.gnome.org/action/show//GnomeOS?action=show&redirect=Projects%2FGnomeContinuous) - это то место, где возник проект OSTree - как высокопроизводительная система непрерывной доставки / тестирования для GNOME. 

[Liri OS](https://liri.io/download/silverblue/) под Fedora Silverblue поддерживает возможность установить свой дистрибутив с помощью ostree. 

[TorizonCore](https://developer.toradex.com/knowledge-base/torizoncore-overview) использует libostree и 
Aktualizr в качестве основы для обновлений OTA с совместимых платформ, 
включая [Torizon OTA](https://developer.toradex.com/knowledge-base/torizon-update-system). 

## Инструменты сборки дистрибутива

[meta-updater](https://github.com/advancedtelematic/meta-updater) - это слой, доступный для [OpenEmbedded](http://www.openembedded.org/wiki/Main_Page)-систем.

[QtOTA](https://doc.qt.io/archives/QtOTA/) - это среда беспроводного обновления Qt, использующая libostree.

Инструмент сборки и интеграции [BuildStream](https://github.com/apache/buildstream/) поддерживает импорт и экспорт из репозиториев libostree.

Fedora [coreos-assemblyr](https://github.com/coreos/coreos-assembler) - это инструмент сборки, используемый для создания производных от Fedora CoreOS. 

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
