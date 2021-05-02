# Anatomy of an OSTree repository

    Core object types and data model
        Commit objects
        Dirtree objects
        Dirmeta objects
        Content objects
    Repository types and locations
        Refs
        The summary file
            Licensing for this document:
## Основные типы объектов и модель данных

Основные идеи и модели OSTree позаимствованы из git;.
Стоит потратить некоторое время на то, чтобы ознакомиться с [Git Internals](http://git-scm.com/book/en/v2), так как этот раздел предполагает некоторые знания о том, как работает git.

Типы объектов OSTree похожи на git; у него есть объекты фиксации (commit) и объекты содержимого (content). 
В Git есть объекты-каталогов, тогда как OSTree разбивает их на «dirtree» и «dirnmeta». 
В  отличие от git, контрольные суммы OSTree - это SHA256. 
И что наиболее важно, его объекты содержимого включают атрибуты uid, gid и extended (но без timestamps). 


### Объекты коммит

Коммит-объект  содержит метаданные, такие как временная метка, сообщение журнала и, что наиболее важно, ссылку на  dirtree/dirmeta c контрольными суммами, которые описывают корневой каталог файловой системы. 
Также, как и в git, у каждого коммита в OSTree может быть родитель. 
Он предназначен для хранения истории ваших двоичных сборок, точно так же, как git хранит историю управления версиями. 
Однако OSTree также упрощает удаление данных при условии, что вы можете восстановить их из исходного кода. 

### Объекты Dirtree

dirtree содержит отсортированный массив пар (имя файла, контрольная сумма) для контент-объектов  и второй отсортированный массив (имя файла, контрольная сумма dirtree, контрольная сумма dirmeta), которые являются подкаталогами. Этот тип объектов хранится в виде файлов с расширением `*.dirtree` в каталоге объектов. 

### Объекты Dirmeta
В git древовидные объекты содержат метаданные, такие как права доступа для своих потомков. 
Но OSTree разбивает это на отдельный объект, чтобы избежать дублирования расширенных списков атрибутов. 
Эти типы объектов хранятся в виде файлов с суффиксом `*.dirmeta` в каталоге объектов. 

### Объекты контента

В отличие от первых трех типов объектов, которые представляют собой метаданные, предназначенные для использования с помощью `mmap()`, объект содержимого имеет отдельные разделы внутреннего заголовка и полезной нагрузки. Заголовок содержит uid, gid, mode и цель символической ссылки (для символических ссылок), а также расширенные атрибуты. После заголовка для обычных файлов следует содержимое. Эти части вместе образуют хэш SHA256 для объектов содержимого. Объекты типа контента в этом формате существуют только в архивных репозиториях OSTree. Сегодня часть содержимого заархивирована с помощью gzip, а объекты хранятся в виде файлов, оканчивающихся на .filez, в каталоге объектов. Поскольку хэш SHA256 формируется на несжатом содержимом, эти файлы не соответствуют хэшу, которым они названы.

Формат данных OSTree намеренно не содержит отметок времени (timestamps). Причина в том, что файлы данных могут быть загружены в разное время и разными системами сборки, поэтому они будут иметь разные временные метки, но идентичное физическое содержимое. Эти файлы могут быть большими, поэтому большинство пользователей хотели бы, чтобы они были общими как в репозитории, так и между репозиторием и развертываниями.

Это может вызвать проблемы с программами, которые проверяют, не устарели ли файлы, сравнивая временные метки. Для Git логичный выбор - не связываться с временными метками, потому что ненужное перестроение лучше, чем сломанное дерево. Однако OSTree привязывают файлы через жестские ссылки (hardlinks), чтобы проверить их, и предполагается, что коммиты внутренне согласованы и не требуют этапов сборки. 
По этой причине OSTree действует так, как будто для всех временных меток установлено значение time_t 0, поэтому сравнения будут считаться актуальными. 
Обратите внимание, что в некоторых выпусках OSTree использовал 1 для исправления предупреждений, таких как GNU Tar, выдающий «неправдоподобно старую отметку времени» с 0; однако, пока у нас не будет механизма для точного перехода к 1, для совместимости OSTree снова будет использовать ноль.

## Repository types and locations

Also unlike git, an OSTree repository can be in one of four separate modes: bare, bare-user, bare-user-only, and archive. A bare repository is one where content files are just stored as regular files; it’s designed to be the source of a “hardlink farm”, where each operating system checkout is merely links into it. If you want to store files owned by e.g. root in this mode, you must run OSTree as root.

The bare-user mode is a later addition that is like bare in that files are unpacked, but it can (and should generally) be created as non-root. In this mode, extended metadata such as owner uid, gid, and extended attributes are stored in extended attributes under the name user.ostreemeta but not actually applied. The bare-user mode is useful for build systems that run as non-root but want to generate root-owned content, as well as non-root container systems.

The bare-user-only mode is a variant to the bare-user mode. Unlike bare-user, neither ownership nor extended attributes are stored. These repos are meant to to be checked out in user mode (with the -U flag), where this information is not applied anyway. Hence this mode may lose metadata. The main advantage of bare-user-only is that repos can be stored on filesystems which do not support extended attributes, such as tmpfs.

In contrast, the archive mode is designed for serving via plain HTTP. Like tar files, it can be read/written by non-root users.

On an OSTree-deployed system, the “system repository” is /ostree/repo. It can be read by any uid, but only written by root. The ostree command will by default operate on the system repository; you may provide the --repo argument to override this, or set the $OSTREE_REPO environment variable.

### Refs

Like git, OSTree uses the terminology “references” (abbreviated “refs”) which are text files that name (refer to) particular commits. See the Git Documentation for information on how git uses them. Unlike git though, it doesn’t usually make sense to have a “master” branch. There is a convention for references in OSTree that looks like this: exampleos/buildmaster/x86_64-runtime and exampleos/buildmaster/x86_64-devel-debug. These two refs point to two different generated filesystem trees. In this example, the “runtime” tree contains just enough to run a basic system, and “devel-debug” contains all of the developer tools and debuginfo.

The ostree supports a simple syntax using the caret ^ to refer to the parent of a given commit. For example, exampleos/buildmaster/x86_64-runtime^ refers to the previous build, and exampleos/buildmaster/x86_64-runtime^^ refers to the one before that.

### The summary file

A later addition to OSTree is the concept of a “summary” file, created via the ostree summary -u command. This was introduced for a few reasons. A primary use case is to be compatible with Metalink, which requires a single file with a known checksum as a target.

The summary file primarily contains two mappings:

-    A mapping of the refs and their checksums, equivalent to fetching the ref file individually
-    A list of all static deltas, along with their metadata checksums

This currently means that it grows linearly with both items. On the other hand, using the summary file, a client can enumerate branches.

Further, fetching the summary file over e.g. pinned TLS creates a strong end-to-end verification of the commit or static delta.

The summary file can also be GPG signed (detached). This is currently the only way to provide GPG signatures (transitively) on deltas.

If a repository administrator creates a summary file, they must thereafter run ostree summary -u to update it whenever a ref is updated or a static delta is generated.
