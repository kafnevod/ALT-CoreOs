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

Коммит-объект  содержит метаданные, такие как временная метка, сообщение журнала и, что наиболее важно, ссылку на  dirtree/dirmeta c контрольными суммами, которые описывают корневой каталог файловой си### Объекты Dirtree

dirtree содержит отсортированный массив пар (имя файла, контрольная сумма) для контент-объектов  и второй отсортированный массив (имя файла, контрольная сумма dirtree, контрольная сумма dirmeta), которые являются подкаталогами. Этот тип объектов хранится в виде файлов с расширением .dirtree в каталоге объектов. стемы. 
Также, как и в git, у каждого коммита в OSTree может быть родитель. 
Он предназначен для хранения истории ваших двоичных сборок, точно так же, как git хранит историю управления версиями. 
Однако OSTree также упрощает удаление данных при условии, что вы можете восстановить их из исходного кода. 

### Объекты Dirtree

dirtree содержит отсортированный массив пар (имя файла, контрольная сумма) для контент-объектов  и второй отсортированный массив (имя файла, контрольная сумма dirtree, контрольная сумма dirmeta), которые являются подкаталогами. Этот тип объектов хранится в виде файлов с расширением .dirtree в каталоге объектов. 

### Объекты Dirmeta
В git древовидные объекты содержат метаданные, такие как права доступа для своих потомков. 
Но OSTree разбивает это на отдельный объект, чтобы избежать дублирования расширенных списков атрибутов. 
Эти типы объектов хранятся в виде файлов с суффиксом *.dirmeta в каталоге объектов. 
In git, tree objects contain the metadata such as permissions for their children. But OSTree splits this into a separate object to avoid duplicating extended attribute listings. These type of objects are stored as files ending with .dirmeta in the objects directory.

### Content objects

Unlike the first three object types which are metadata, designed to be mmap()ed, the content object has a separate internal header and payload sections. The header contains uid, gid, mode, and symbolic link target (for symlinks), as well as extended attributes. After the header, for regular files, the content follows. These parts together form the SHA256 hash for content objects. The content type objects in this format exist only in archive OSTree repositories. Today the content part is gzip’ed and the objects are stored as files ending with .filez in the objects directory. Because the SHA256 hash is formed over the uncompressed content, these files do not match the hash they are named as.

The OSTree data format intentionally does not contain timestamps. The reasoning is that data files may be downloaded at different times, and by different build systems, and so will have different timestamps but identical physical content. These files may be large, so most users would like them to be shared, both in the repository and between the repository and deployments.

This could cause problems with programs that check if files are out-of-date by comparing timestamps. For Git, the logical choice is to not mess with timestamps, because unnecessary rebuilding is better than a broken tree. However, OSTree has to hardlink files to check them out, and commits are assumed to be internally consistent with no build steps needed. For this reason, OSTree acts as though all timestamps are set to time_t 0, so that comparisons will be considered up-to-date. Note that for a few releases, OSTree used 1 to fix warnings such as GNU Tar emitting “implausibly old time stamp” with 0; however, until we have a mechanism to transition cleanly to 1, for compatibilty OSTree is reverted to use zero again.

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
