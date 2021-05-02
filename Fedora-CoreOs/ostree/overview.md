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

## Сравнение с «менеджерами пакетов» rpm, dpkg, ...

Поскольку OSTree разработан для развертывания основных операционных систем, сравнение с традиционными «менеджерами пакетов», такими как dpkg и rpm, является показательным. Пакеты традиционно состоят из частичных деревьев файловых систем с прикрепленными метаданными и скриптами, которые динамически собираются на клиентском компьютере после процесса разрешения зависимостей.

Напротив, OSTree поддерживает только запись и развертывание полных (загрузочных) деревьев файловых систем. Он не имеет встроенных знаний о том, как было сгенерировано данное дерево файловой системы, или о происхождении отдельных файлов, или о зависимостях, и о описаниях отдельных компонентов. Другими словами, OSTree занимается только доставкой и развертыванием; вы, вероятно, по-прежнему захотите включить в каждое дерево метаданные об отдельных компонентах, которые вошли в дерево. Например, системный администратор может захотеть узнать, какая версия OpenSSL была включена в ваше дерево, поэтому вы должны поддерживать эквивалент rpm -q (*в Fedora CoreOs это rpm-ostree*) или dpkg -L.

Ядро OSTree делает упор на репликацию деревьев ОС, доступных только для чтения, через HTTP, и где ОС включает (при желании) полностью отдельный механизм для установки приложений, хранящихся в / var, если они являются глобальными для системы, или / home для установки приложений для каждого пользователя. . Пример механизма приложения - http://docker.io/

Однако вполне возможно использовать OSTree под системой пакетов, где содержимое /usr вычисляется на клиенте. Например, при установке пакета вместо изменения текущей файловой системы диспетчер пакетов может собрать новое дерево файловой системы, которое накладывает новые пакеты поверх базового дерева, записать его в локальный репозиторий OSTree, а затем настроить его. для следующей загрузки (*реализовано в rmv-ostree Fedora Core*). Для поддержки этой модели OSTree предоставляет (интроспективную) разделяемую библиотеку C. 


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

