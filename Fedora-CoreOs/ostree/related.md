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

## Объединение dpkg / rpm + (BTRFS / LVM)

В этом подходе в рамках системного менеджера пакетов используется инструмент для создания моментальных снимков блоков/файловой_системы.

Инструмент [oVirt Node](https://ru.wikipedia.org/wiki/OVirt), работающий с образами, является примером этого подхода, как и некоторые другие ниже.

Касательно BTRFS как иструмента получения мгнрвенных снимков файловой системы - автор OSTree считает, что хранилище Linux - это обширный мир, и, 
хотя BTRFS неплох, он не везде исползуется сейчас и не будет везде в ближайшем будущем. 
Есть и другие недавно разработанные файловые системы, такие как [f2fs](https://en.wikipedia.org/wiki/F2FS), и Red Hat Enterprise Linux по-прежнему использует [XFS](https://en.wikipedia.org/wiki/XFS).

Использование инструмента создания снимков под менеджером пакетов действительно помогает. 
В остальной части этого текста мы будем использовать «BTRFS» как основной универсальный инструмент для получения моментальных снимков файловой системы.

Очевидно, что нужно расположить BTRFS под dpkg/rpm и создать отдельный вложенный том для /home, чтобы при откате не терялись ваши данные. 
См., Например, [Fedora BTRFS Rollback Feature](https://fedoraproject.org/wiki/Features/SystemRollbackWithBtrfs).

В более общем плане, если вы хотите использовать BTRFS для отката изменений, сделанных dpkg/rpm, вы должны тщательно настроить макет раздела, 
чтобы файлы, размещенные dpkg/rpm, устанавливались в подтоме для создания моментального снимка.

Эта проблема во многих отношениях решается с помощью изменений OSTree, таких как размещение всего локального состояния в /var (например, /usr/local -> /var/usrlocal). 
Затем можно сделать снимок BTRFS /usr. 

В общем, если кто-то действительно пытается конкретизировать подход BTRFS, потребуется нетривиальный промежуточный уровень кода между dpkg/rpm и BTRFS (или глубокая осведомленность о BTRFS в самом dpkg/rpm). Хорошим примером этого является проект [snapper.io](http://snapper.io/).

Автор OSTree считает, что для операционных систем общего назначения лучше иметь полную свободу на уровне блочного хранилища. 
Например, весьма полезна возможность выбора dm-crypt для каждого развертывания; не каждый сайт хочет терять производительность. Можно выбрать LVM или нет и т. д.

Там, где это применимо, OSTree использует возможности copy-on-write/reflink, предлагаемые ядром для / etc. 
Он использует  ioctl(FICLONE) и copy_file_range().

Еще одно важное различие между использованием OSTree по умолчанию и менеджерами пакетов заключается в том, являются ли обновления по умолчанию «интерактивными» или «автономными». Дизайн OSTree по умолчанию записывает обновления в новый корень, оставляя работающую систему без изменений. Это означает, что подготовка обновлений осуществляется без прерывания работы и является безопасной - если системе не хватает места на диске, его легко восстановить. 
Однако в проекте rpm-ostree также ведется работа по поддержке онлайн-обновлений.

OSTree поддерживает использование репозиториев «bare-user», для использования которых не требуется root. 
Использование уровня файловой системы без root сложнее и, вероятно, потребует setuid helper или привилегированного сервиса.

Наконец, посмотрите следующую часть ChromiumOS, чтобы узнать, почему гибридная, но интегрированная система пакетов/образов улучшает это. 

## ChromiumOS updater

Многие люди, которые смотрят на OSTree, больше всего заинтересованы в использовании его в качестве средства обновления для встроенных или фиксированных систем, 
аналогично вариантам использования средства обновления ChromiumOS.

Подход ChromiumOS использует два раздела, которые меняются местами через загрузчик. 
Он имеет очень эффективный для сети протокол обновления, использующий настраиваемую двоичную дельта-схему между моментальными снимками файловой системы.

Эта модель даже позволяет переключать типы файловой системы в обновлении.

Основным недостатком этого подхода является то, что размер ОС на диске всегда увеличивается вдвое. 
В отличие от этого, OSTree использует простые жесткие ссылки Unix, что означает, что по существу требуется только дисковое пространство, пропорциональное измененным файлам, плюс небольшие фиксированные накладные расходы.

Это означает, что с OSTree можно легко иметь более двух деревьев (развертываний). 
Другой пример:  репозиторий OSTree также может использоваться для контейнеров приложений.

Наконец, автор OSTree считает, что во многих случаях действительно требуется гибридная модель - репликация образов с возможностью наложения на них  некоторых дополнительных компонент (например, пакетов). 
Именно это и стремится поддерживать rpm-ostree. 

# Ubuntu Image Based Updates

См. https://wiki.ubuntu.com/ImageBasedUpgrades. 
Архитектурно очень похоже на ChromeOS, хотя более интересным является обсуждение поддержки установки пакетов поверх, аналогично иерархии пакетов rpm-ostree.

## Clear Linux Software update

Система обновления программного обеспечения [Clear Linux](https://clearlinux.org/) не очень хорошо документирована. 
В [этом списке рассылки](https://lists.clearlinux.org/hyperkitty/) есть некоторая переработанная проектная документация.

Как и статические дельты OSTree, он также использует bsdiff для повышения эффективности сетевых обменов данными.

Со временем сюда будет добавлена ​​дополнительная информация. 
Автор OSTree считает, что на данный момент «средство обновления CL» не является полностью атомарным в том смысле, 
что, поскольку система применяет обновления в реальном времени, существует окно, в котором корень ОС может быть несовместимым.

## casync

Проект [systemd casync](https://github.com/systemd/casync) относительно новый. 
В настоящее время это скорее библиотека хранения, и она не поддерживает логику более высокого уровня для таких вещей, как подписи GPG, информация о версиях и т. Д. 
Это в основном уровень OstreeRepo. 
Переход на уровень OstreeSysroot - отсутствуют.
Такие вещи, как управление конфигурацией загрузчика и, самое главное, правильное слияние для /etc. не реализованы.
casync также ничего не знает о SELinux.

OSTree на самом деле сегодня является общей библиотекой, причем довольно давно. 
Это упростило создание проектов более высокого уровня, таких как rpm-ostree.

Сегодня основная проблема casync заключается в том, что она не поддерживает сборку мусора на стороне сервера. 
Сборщик мусора OSTree работает симметрично на стороне сервера и клиента.

Вообще говоря, casync - это нечто среднее между двумя подходами и он разделяет их общие недостатки.

## Mender.io

Mender.io - еще одна реализация подхода с двумя разделами. 

## OLPC update

OSTree - это в основном обобщение olpc-update, за исключением использования простого HTTP вместо rsync. 
OSTree имеет понятие отдельных деревьев, которые можно отслеживать независимо или параллельно, 
но при этом совместно использовать хранилище через репозиторий с жесткой связью, тогда как olpc-update использует номера версий для одной ОС.

OSTree имеет встроенную простую репликацию HTTP, которая может обслуживаться со статического веб-сервера, 
тогда как olpc-update использует rsync (больше нагрузки на сервер, но более эффективно на сетевой стороне). 
Решение OSTree для улучшения пропускной способности сети - это статические дельты.

См. [Этот комментарий](https://blog.verbum.org/2013/08/26/ostree-v2013-6-released/#comment-1169) для сравнения. 

## NixOS / Nix

См. [NixOS](https://nixos.org/). Этоn проект очень похож на OSTree. 
И NixOS, и OSTree поддерживают идею независимых загрузочных «корневых файловых систем».

В NixOS доступ к файлам в пакете осуществляется по пути, зависящему от контрольных сумм входных данных пакета (зависимости сборки) - см. 
[Магазин Nix](https://nixos.org/manual/nix/stable/#chap-package-management/). 
Однако OSTree использует модель commit/deploy - она ​​не привязана к какому-либо конкретному макету каталога, и вы можете поместить любые данные в OSTree, например, стандартный макет FHS. 
Как положительный, так и отрицательный момент модели Nix заключается в том, что изменение зависимостей сборки (например, сборка с использованием более новой версии gcc) требует каскадной перестройки всего. 
Это хорошо, потому что позволяет легко вносить масштабные общесистемные изменения, такие как обновления gcc, и позволяет одновременно устанавливать несколько версий пакетов. 
Однако обновление безопасности, например, glibc вынуждает перестраивать все с нуля, поэтому Nix непрактичен в определнных условиях. 
OSTree поддерживает использование системы сборки, которая просто перестраивает отдельные компоненты (пакеты) по мере их изменения без принудительного перестроения их зависимостей.

Nix автоматически определяет зависимости пакетов во время выполнения, сканируя содержимое на предмет хэшей. 
OSTree поддерживает только образы системного уровня и не управляет зависимостями. 
Nix может хранить произвольные файлы с помощью `nix-store –add`, но чаще всего пути добавляются в результате запуска производного файла, созданного с использованием языка Nix. 
OSTree не зависит от системы сборки; деревья файловых систем коммитятся с использованием простого C API, и это единственный способ закомиттить файлы.

OSTree автоматически разделяет хранилище идентичных данных с помощью жестких ссылок. 
Nix также может выполнять дедупликацию с использованием жестких ссылок, используя параметр `auto-optimize-store`, но он не включен по умолчанию. 
OSTree предоставляет интерфейс командной строки, похожий на git, для просмотра хранилища с адресным содержимым, в то время как Nix не имеет этой функции.

Nix использовал неизменяемый бит для предотвращения изменений в `/nix/store`, но теперь он использует монтирование привязки только для чтения. 
Bind mount может быть перемонтирован в частном порядке, что обеспечивает привилегированный доступ для записи для каждого процесса. 
OSTree использует неизменяемый бит в корне развертывания и монтирует /usr как доступный только для чтения.

NixOS поддерживает переключение образов ОС на лету, поддерживая корни загруженной системы и текущей системы. Неясно, насколько хорошо работает этот подход. OSTree в настоящее время требует перезагрузки для переключения образов.

Наконец, NixOS поддерживает установку пользовательских пакетов из доверенных репозиториев без необходимости получения прав root, используя доверенный демон. 
Flatpak, основанный на OSTree, также имеет системный помощник на основе набора политик, который позволяет вам аутентифицироваться через polkit для установки в системный репозиторий. 

## Solaris IPS

См. [Solaris IPS](https://github.com/oracle/solaris-ips). 
В целом, это дизайн, аналогичный комбинации BTRFS+RPM/deb. Есть система управления загрузчиком, которая комбинируется со снимками. Он относительно хорошо продуман, но представляет собой сборку системы на стороне клиента. Если кто-то хочет создавать образы серверов и надежно реплицировать их, это будет другая система.

## Серверы Google (индивидуальный подход, подобный rsync, обновления в реальном времени)

В этом документе рассказывается о том, как Google (по крайней мере, в один момент) управлял обновлениями для хост-систем для некоторых серверов: 
[Live Upgrading Thousands of Servers from an Ancient Red Hat Distribution to 10 Year Newer Debian Based One](https://www.usenix.org/conference/lisa13/technical-sessions/presentation/merlin).

## Конари

См. [Conary](https://github.com/sassoftware/conary). 
Если rpm/dpkg похожи на CVS, Conary ближе к Subversion. Это неплохо, но например его модель отката скорее спонтанная, а не атомарная. Это также полностью клиентская система и не имеет репликации, подобной изображению, с дельтами.

# bmap
См. [Bmap](https://source.tizen.org/documentation/reference/bmaptool/introduction). 
Инструмент для оптимизированного копирования образов дисков. Предназначен для использования в автономном режиме, поэтому напрямую не сопоставим.

## Git

Хотя OSTree был назван «Git for Binaries», и оба они разделяют идею хранилища хешированного содержимого, детали реализации совершенно разные. 
OSTree поддерживает расширенные атрибуты и использует SHA256 вместо SHA1 Git. 
Он “checks out” файлы через жесткие ссылки, а не копирует, и поэтому требует, чтобы checksout был неизменным. 
На данный момент коммиты OSTree могут иметь не более одного родителя, в отличие от Git, который допускает произвольное число. 
Git использует протокол smart-delta для обновлений, в то время как OSTree использует один HTTP-запрос на каждый измененный файл или может генерировать статические дельты. 

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
