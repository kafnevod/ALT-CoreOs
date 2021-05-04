# Writing a buildsystem and managing repositories

    Build vs buy
    Initializing
    Writing your own OSTree buildsystem
    Constructing trees from unions
    Migrating content between repositories
    More sophisticated repository management
        Licensing for this document:

OSTree - это не пакетная система. Он не поддерживает напрямую исходный код сборки. Скорее, это инструмент для транспортировки и управления контентом, наряду с независимыми от системы аспектами, такими как управление загрузчиком при обновлениях.

Предположим, что мы планируем генерировать коммиты на сервере сборки, а затем заставляем клиентские системы реплицировать их. Выполнение сборки на стороне клиента, конечно, также возможно, но это обсуждение будет сосредоточено в первую очередь на проблемах на стороне сервера.

## Построить самому или покупки

Следовательно, вам нужно либо выбрать существующий инструмент для записи контента в репозиторий OSTree, либо написать свой собственный. 
Примером существующего инструмента является rpm-ostree - он принимает в качестве входных список RPM-пакетов и фиксирует их (в настоящее время он ориентирован на серверную сторону, но также возможно его использлвание и  на стороне клиента). 

Предполагаем, что у вас есть одно хранилище архивов:
```
mkdir репо
ostree --repo=repo init --mode=archive
```

Вы можете экспортировать это через статический веб-сервер и настроить клиентов на загрузки дерева с него. 

## Написание собственной системы сборки OSTree

Существует очень много систем, которые в основном следуют шаблону:
```
$pkg --installroot=/path/to/tmpdir install foo bar baz
$imagesystem commit --root=/path/to/tmpdir
```

Для различных значений `$pkg`, таких как `yum`, `apt-get` и т. д., 
и значениями `$imagesystem` могут быть простые архивы tar, образы машин Amazon, ISO и т. Д.

Теперь поговорим о ситуации, когда $imagesystem - это OSTree. Общая идея OSTree заключается в том, что где бы вы ни хранили серию архивов приложений или образов ОС, OSTree, 
очевидно, будет лучшим решением. 
Например, он поддерживает подписи GPG, двоичные дельты, запись конфигурации загрузчика и т. д.

OSTree не включает систему сборки пакетов / компонентов просто потому, что уже существует множество хороших систем - скорее, он предназначен для обеспечения поддержки уровня инфраструктуры.

Вышеупомянутый менеджер компоновки rpm-ostree выбирает RPM в качестве значения `$pkg` - поэтому двоичные файлы создаются как RPM, а затем передаются в целом в коммит OSTree.

Но давайте поговорим о создании менеджера пакетов. 
Если вы просто экспериментируете, довольно легко начать с командной строки. Для этого предположим, что у вас есть процесс сборки, который выводит дерево каталогов - 
мы назовем этот инструмент $pkginstallroot (это может быть yum --installroot или debootstrap и т. д.).

Ваш первоначальный прототип будет выглядеть так:
```
$pkginstallroot /path/to/tmpdir
ostree --repo=repo commit -s 'build' -b exampleos/x86_64/standard --tree=dir=/path/to/tmpdir
```

В качестве альтернативы, если ваша система сборки может создавать архив, вы можете зафиксировать этот архив в OSTree. Например, OpenEmbedded может выводить архив, и его можно зафиксировать с помощью:
```
ostree commit -s 'build' -b exampleos/x86_64/standard --tree=tar=myos.tar
```
## Constructing trees from unions

The above is a very simplistic model, and you will quickly notice that it’s slow. This is because OSTree has to re-checksum and recompress the content each time it’s committed. (Most of the CPU time is spent in compression which gets thrown away if the content turns out to be already stored).

A more advanced approach is to store components in OSTree itself, then union them, and recommit them. At this point, we recommend taking a look at the OSTree API, and choose a programming language supported by GObject Introspection to write your buildsystem scripts. Python may be a good choice, or you could choose custom C code, etc.

For the purposes of this tutorial we will use shell script, but it’s strongly recommended to choose a real programming language for your build system.

Let’s say that your build system produces separate artifacts (whether those are RPMs, zip files, or whatever). These artifacts should be the result of make install DESTDIR= or similar. Basically equivalent to RPMs/debs.

Further, in order to make things fast, we will need a separate bare-user repository in order to perform checkouts quickly via hardlinks. We’ll then export content into the archive repository for use by client systems.
```
mkdir build-repo
ostree --repo=build-repo init --mode=bare-user
```

You can begin committing those as individual branches:
```
ostree --repo=build-repo commit -b exampleos/x86_64/bash --tree=tar=bash-4.2-bin.tar.gz
ostree --repo=build-repo commit -b exampleos/x86_64/systemd --tree=tar=systemd-224-bin.tar.gz
```

Set things up so that whenever a package changes, you redo the commit with the new package version - conceptually, the branch tracks the individual package versions over time, and defaults to “latest”. This isn’t required - one could also include the version in the branch name, and have metadata outside to determine “latest” (or the desired version).

Now, to construct our final tree:
```
rm -rf exampleos-build
for package in bash systemd; do
  ostree --repo=build-repo checkout -U --union exampleos/x86_64/${package} exampleos-build
done
# Set up a "rofiles-fuse" mount point; this ensures that any processes
# we run for post-processing of the tree don't corrupt the hardlinks.
mkdir -p mnt
rofiles-fuse exampleos-build mnt
# Now run global "triggers", generate cache files:
ldconfig -r mnt
  (Insert other programs here)
fusermount -u mnt
ostree --repo=build-repo commit -b exampleos/x86_64/standard --link-checkout-speedup exampleos-build
```

There are a number of interesting things going on here. The major architectural change is that we’re using --link-checkout-speedup. This is a way to tell OSTree that our checkout is made via hardlinks, and to scan the repository in order to build up a reverse (device, inode) -> checksum mapping.

In order for this mapping to be accurate, we needed the rofiles-fuse to ensure that any changed files had new inodes (and hence a new checksum).

## Migrating content between repositories

Now that we have content in our build-repo repository (in bare-user mode), we need to move the exampleos/x86_64/standard branch content into the repository just named repo (in archive mode) for export, which will involve zlib compression of new objects. We likely want to generate static deltas after that as well.

Let’s copy the content:
```
ostree --repo=repo pull-local build-repo exampleos/x86_64/standard
```

Clients can now incrementally download new objects - however, this would also be a good time to generate a delta from the previous commit.
```
ostree --repo=repo static-delta generate exampleos/x86_64/standard
```

## More sophisticated repository management

Next, see Repository Management for the next steps in managing content in OSTree repositories.
