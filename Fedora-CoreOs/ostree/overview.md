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

## Сравнение с репликацией блоков / образов (image based)

OSTree имеет некоторое сходство с «тупой» репликацией и развертываниями без сохранения состояния, например с моделью, распространенной в «облачных» развертываниях, где узлы загружаются с диска (фактически) только для чтения, а пользовательские данные хранятся на разных томах. Преимущество такой репликации, реализованной как в OSTree, так и в облачной модели, состоит в том, что она надежна и предсказуема.

Но в отличие от многих image based развертываний,  OSTree поддерживает ровно два постоянных каталога с возможностью записи, которые сохраняются при обновлении: `/etc` и `/var`.

Поскольку OSTree работает на уровне файловой системы Unix, он работает поверх любой файловой системы или структуры блочного хранилища; можно реплицировать заданное дерево файловой системы из репозитория OSTree в обычный *ext4*, BTRFS, XFS (*реализовано в Fedora Core Os*) или вообще любую Unix-совместимую файловую систему, которая поддерживает жесткие (hard) ссылки. 

> Примечание. Для файловой системы BTRFS OSTree поддерживает дополнительные функции, реализованные в BTRFS.

OSTree ортогонален механизмам виртуализации на основе образов виртуальных машин типа AMI и qcow2.
Его преимущаство в том, что он позволает настраивать развернутые на узлах стандартные образы сгенерированные для каждого узла.
не загружая полностью AMI или qcow2 образы сгенерированные   

На практике для пользователей конфигураций с «голым железом» (metal bare) наиболее полезной будет модель OSTree. 

## Atomic transitions between parallel-installable read-only filesystem trees

Another deeply fundamental difference between both package managers and image-based replication is that OSTree is designed to parallel-install multiple versions of multiple independent operating systems. OSTree relies on a new toplevel ostree directory; it can in fact parallel install inside an existing OS or distribution occupying the physical / root.

On each client machine, there is an OSTree repository stored in /ostree/repo, and a set of “deployments” stored in /ostree/deploy/$STATEROOT/$CHECKSUM. Each deployment is primarily composed of a set of hardlinks into the repository. This means each version is deduplicated; an upgrade process only costs disk space proportional to the new files, plus some constant overhead.

The model OSTree emphasizes is that the OS read-only content is kept in the classic Unix /usr; it comes with code to create a Linux read-only bind mount to prevent inadvertent corruption. There is exactly one /var writable directory shared between each deployment for a given OS. The OSTree core code does not touch content in this directory; it is up to the code in each operating system for how to manage and upgrade state.

Finally, each deployment has its own writable copy of the configuration store /etc. On upgrade, OSTree will perform a basic 3-way diff, and apply any local changes to the new copy, while leaving the old untouched.

