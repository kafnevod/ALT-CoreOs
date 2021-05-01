# OSTree Overview

    Introduction
    Hello World example
    Comparison with “package managers”
    Comparison with block/image replication
    Atomic transitions between parallel-installable read-only filesystem trees
        Licensing for this document:
## Введение

OSTree - это система обновления для операционных систем на базе Linux, которая выполняет атомарное обновление полных деревьев файловых систем. Это не пакетная система; скорее, он предназначен для их дополнения. Основная модель - это составление пакетов на сервере, а затем их репликация  клиентам.

Базовую архитектуру можно кратко охарактеризовать как «git для двоичных файлов операционной системы». Он работает в пользовательском пространстве и будет работать поверх любой файловой системы Linux. По своей сути, это хранилище объектов с адресацией к содержимому, похожее на git, с ветвями (branches) (или «refs») для отслеживания значимых деревьев файловых систем в хранилище. Точно так же можно проверить или зафиксировать (commit) эти ветки.

На вершине этого уровня находится конфигурация загрузчика, управление / и т.д., а также другие функции для выполнения обновления, помимо репликации файлов.

Вы можете использовать OSTree автономно в чистой модели репликации, но другой подход заключается в добавлении диспетчера пакетов поверх, создавая таким образом гибридную систему дерева / пакетов. 


## Пример Hello World

OSTree в основном используется как библиотека, но краткий обзор использования его инструментов CLI может дать общее представление о том, как он работает на самом базовом уровне.

Вы можете создать новый репозиторий OSTree с помощью init: 
```
$ ostree --repo=repo init
```

Это создаст новый каталог repo, содержащий ваш репозиторий. 
Теперь давайте подготовим данные для добавления в репо: 
```
$ mkdir tree
$ echo "Hello world!" > tree/hello.txt
```
Теперь мы можем импортировать наше дерево / каталог с помощью команды commit: 
```
$ ostree --repo=repo commit --branch=foo tree/
```
Это создаст новую ветку foo, указывающую на полное дерево, импортированное из tree /. Фактически, теперь мы могли удалить дерево /, если бы захотели.

Чтобы убедиться, что у нас действительно есть ветка foo, вы можете использовать команду refs: 
```
$ ostree --repo=repo refs
foo
```

Мы также можем просмотреть дерево файловой системы с помощью команд ls и cat: 
```
$ ostree --repo=repo ls foo
d00775 1000 1000      0 /
-00664 1000 1000     13 /hello.txt
$ ostree --repo=repo cat foo /hello.txt
Hello world!
```

И, наконец, мы можем проверить наше дерево из репозитория: 
```
$ ostree --repo=repo checkout foo tree-checkout/
$ cat tree-checkout/hello.txt
Hello world!
```

## Comparison with “package managers”

Because OSTree is designed for deploying core operating systems, a comparison with traditional “package managers” such as dpkg and rpm is illustrative. Packages are traditionally composed of partial filesystem trees with metadata and scripts attached, and these are dynamically assembled on the client machine, after a process of dependency resolution.

In contrast, OSTree only supports recording and deploying complete (bootable) filesystem trees. It has no built-in knowledge of how a given filesystem tree was generated or the origin of individual files, or dependencies, descriptions of individual components. Put another way, OSTree only handles delivery and deployment; you will likely still want to include inside each tree metadata about the individual components that went into the tree. For example, a system administrator may want to know what version of OpenSSL was included in your tree, so you should support the equivalent of rpm -q or dpkg -L.

The OSTree core emphasizes replicating read-only OS trees via HTTP, and where the OS includes (if desired) an entirely separate mechanism to install applications, stored in /var if they’re system global, or /home for per-user application installation. An example application mechanism is http://docker.io/

However, it is entirely possible to use OSTree underneath a package system, where the contents of /usr are computed on the client. For example, when installing a package, rather than changing the currently running filesystem, the package manager could assemble a new filesystem tree that layers the new packages on top of a base tree, record it in the local OSTree repository, and then set it up for the next boot. To support this model, OSTree provides an (introspectable) C shared library.

## Comparison with block/image replication

OSTree shares some similarity with “dumb” replication and stateless deployments, such as the model common in “cloud” deployments where nodes are booted from an (effectively) readonly disk, and user data is kept on a different volumes. The advantage of “dumb” replication, shared by both OSTree and the cloud model, is that it’s reliable and predictable.

But unlike many default image-based deployments, OSTree supports exactly two persistent writable directories that are preserved across upgrades: /etc and /var.

Because OSTree operates at the Unix filesystem layer, it works on top of any filesystem or block storage layout; it’s possible to replicate a given filesystem tree from an OSTree repository into plain ext4, BTRFS, XFS, or in general any Unix-compatible filesystem that supports hard links. Note: OSTree will transparently take advantage of some BTRFS features if deployed on it.

OSTree is orthogonal to virtualization mechanisms like AMIs and qcow2 images, though it’s most useful though if you plan to update stateful VMs in-place, rather than generating new images.

In practice, users of “bare metal” configurations will find the OSTree model most useful.

## Atomic transitions between parallel-installable read-only filesystem trees

Another deeply fundamental difference between both package managers and image-based replication is that OSTree is designed to parallel-install multiple versions of multiple independent operating systems. OSTree relies on a new toplevel ostree directory; it can in fact parallel install inside an existing OS or distribution occupying the physical / root.

On each client machine, there is an OSTree repository stored in /ostree/repo, and a set of “deployments” stored in /ostree/deploy/$STATEROOT/$CHECKSUM. Each deployment is primarily composed of a set of hardlinks into the repository. This means each version is deduplicated; an upgrade process only costs disk space proportional to the new files, plus some constant overhead.

The model OSTree emphasizes is that the OS read-only content is kept in the classic Unix /usr; it comes with code to create a Linux read-only bind mount to prevent inadvertent corruption. There is exactly one /var writable directory shared between each deployment for a given OS. The OSTree core code does not touch content in this directory; it is up to the code in each operating system for how to manage and upgrade state.

Finally, each deployment has its own writable copy of the configuration store /etc. On upgrade, OSTree will perform a basic 3-way diff, and apply any local changes to the new copy, while leaving the old untouched.

