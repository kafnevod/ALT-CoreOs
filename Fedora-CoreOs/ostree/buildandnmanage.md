# Написание системы сборки и управление репозиториями 

    Build vs buy
    Initializing
    Writing your own OSTree buildsystem
    Constructing trees from unions
    Migrating content between repositories
    More sophisticated repository management
        Licensing for this document:

OSTree - это не пакетная система. Он не поддерживает напрямую исходный код сборки. Скорее, это инструмент для транспортировки и управления контентом, наряду с независимыми от системы аспектами, такими как управление загрузчиком при обновлениях.

Предположим, что мы планируем генерировать коммиты на сервере сборки, а затем заставляем клиентские системы реплицировать их. Выполнение сборки на стороне клиента, конечно, также возможно, но это обсуждение будет сосредоточено в первую очередь на проблемах на стороне сервера.

## Построить самому или заказать

Следовательно, вам нужно либо выбрать существующий инструмент для записи контента в репозиторий OSTree, либо написать свой собственный. 
Примером существующего инструмента является [rpm-ostree](https://github.com/coreos/rpm-ostree) - он принимает в качестве входных список RPM-пакетов и фиксирует их (в настоящее время он ориентирован на серверную сторону, но также возможно его использлвание и  на стороне клиента). 

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

В качестве альтернативы, если ваша система сборки может создавать архив, вы можете зафиксировать этот архив в OSTree. Например, 
[OpenEmbedded](http://www.openembedded.org/wiki/Main_Page) может выводить архив, и его можно зафиксировать с помощью:
```
ostree commit -s 'build' -b exampleos/x86_64/standard --tree=tar=myos.tar
```
## Построение деревьев путем операции объединения

Выше описана очень упрощенная модель, и очевидно, что она медленная. 
Это связано с тем, что OSTree приходится повторно проверять контрольную сумму и повторно сжимать контент каждый раз, когда происходит коммит. 
(Большая часть времени ЦП тратится на сжатие, результат которого отбрасывается, если оказывается, что содержимое уже сохранено).

Более продвинутый подход - хранить компоненты в самом OSTree, затем объединять их и повторно объединять. 
На этом этапе мы рекомендуем взглянуть на OSTree API и выбрать язык программирования, поддерживаемый 
[GObject Introspection](https://wiki.gnome.org/Projects/GObjectIntrospection), для написания сценариев вашей системы сборки. 
Python может быть хорошим выбором, или вы можете написать собственный код C и т. д.

В этом руководстве мы будем использовать сценарий оболочки, но настоятельно рекомендуется выбрать реальный язык программирования для вашей системы сборки.

Допустим, ваша система сборки создает отдельные артефакты (будь то RPM-файлы, zip-файлы или что-то еще). 
Эти артефакты должны быть результатом выполнения команды 
`make install DESTDIR=`
или аналогичного. В основном эквивалент RPMs/debs.

Кроме того, чтобы ускорить процесс, нам понадобится отдельный репозиторий для простых пользователей, чтобы быстро выполнять проверки через жесткие ссылки. 
Затем мы экспортируем контент в архивный репозиторий для использования клиентскими системами.
```
mkdir build-репо
ostree --repo=build-repo init --mode=bare-user
```

Вы можете начать коммитить их как отдельные ветки:
```
ostree --repo=build-repo commit -b exampleos/x86_64/bash --tree=tar=bash-4.2-bin.tar.gz
ostree --repo=build-repo commit -b exampleos/x86_64/systemd --tree=tar=systemd-224-bin.tar.gz
```

Настройте все так, чтобы всякий раз, когда пакет изменяется, вы повторяете фиксацию с новой версией пакета - 
концептуально, ветвь отслеживает отдельные версии пакета с течением времени и по умолчанию имеет значение «последняя». 
Это не обязательно - можно также включить версию в название ветки и иметь внешние метаданные для определения «последней» (или желаемой версии).

Теперь, чтобы построить наше окончательное дерево:
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

Здесь происходит ряд интересных вещей. 
Основное архитектурное изменение заключается в том, что мы используем `--link-checkout-speedup`. 
Это способ сообщить OSTree, что наша проверка выполняется через жесткие ссылки, и просканировать репозиторий, чтобы создать сопоставление`reverse (device, inode) -> checksum mapping`.

Чтобы это сопоставление было точным, нам нужен `rofiles-fuse`, чтобы гарантировать, что любые измененные файлы имеют новые `inodes` (и, следовательно, новую контрольную сумму). 


## Перенос содержимого между репозиториями

Теперь, когда у нас есть контент в нашем репозитории сборки-репозитория (в режиме простого пользователя), нам нужно переместить содержимое ветки exampleos / x86_64 / standard в репозиторий с именем repo (в режиме архива) для экспорта, что будет включать сжатие zlib. новых объектов. Скорее всего, после этого мы захотим сгенерировать статические дельты.

Скопируем контент:
``
ostree --repo=repo pull-local build-repo exampleos/x86_64/standard
``

Теперь клиенты могут постепенно загружать новые объекты - однако это также подходящее время для генерации дельты из предыдущей фиксации.
``
ostree --repo=repo static-delta generate exampleos/x86_64/standard
``

## Более сложное управление репозиторием

Затем см. [Управление репозиториями](./contentmanage.md), чтобы узнать о следующих шагах по управлению контентом в репозиториях OSTree.
