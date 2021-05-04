# Managing content in OSTree repositories

    Mirroring repositories
    Separate development vs release repositories
    Promoting content along OSTree branches - “buildmaster”, “smoketested”
    Promoting content between OSTree repositories
    Derived data - static deltas and the summary file
    Pruning our build and dev repositories
    Generating “scratch” deltas for efficient initial downloads
        Licensing for this document:

Если у вас запущена система сборки и вы действительно хотите, чтобы клиентские системы извлекали контент, вам потребуется «управлении репозиторием».

Инструмент командной строки ostree охватывает некоторые основные функции, но не включает рабочие процессы очень высокого уровня. 
Одна из причин заключается в том, что то, как контент доставляется и управляется, очень специфичен для каждой организации. 
Например, некоторые поставщики содержимого операционной системы могут захотеть интегрироваться с определенной системой уведомления об ошибках при создании коммитов.

В этом разделе мы опишем некоторые высокоуровневые идеи и методы управления контентом в репозиториях OSTree, в основном независимые от какой-либо конкретной модели или инструмента. 
Тем не менее, существует связанный вышестоящий проект [ostree-relng-scripts](https://github.com/ostreedev/ostree-releng-scripts), 
в котором есть несколько скриптов, предназначенных для реализации частей этого документа.

Еще одним примером программного обеспечения, которое может помочь в управлении репозиториями OSTree сегодня, является 
[Pulp Project](https://pulpproject.org/), в котором есть [плагин Pulp OSTree](https://repos.fedorapeople.org/pulp/pulp/stable/2/7Server/src/). 


## Зеркалирование репозиториев

Очень часто возникает желание выполнить полное или частичное зеркалирование, 
в частности, за пределами организации (например, вышестоящий поставщик ОС и пользователь, которому нужен автономный и более быстрый доступ к контенту). 
OSTree поддерживает как полное, так и частичное зеркальное отображение содержимого базового архива, но пока не поддерживает зеркалирование  не статических дельт.

Чтобы создать зеркало, сначала создайте архивный репозиторий (вам не нужно запускать его как root), 
затем добавьте `upstream` как удаленный, затем используйте `pull --mirror`.
```
ostree --repo=repo init --mode=archive
ostree --repo=repo remote add exampleos https://exampleos.com/ostree/repo
ostree --repo=repo pull --mirror exampleos:exampleos/x86_64/standard
```

Вы можете использовать параметр `--depth=-1`, чтобы получить всю историю, или положительное целое число, например `3`, чтобы получить только последние `3` коммита.

См. также сценарий `rsync-repos` в [ostree-relng-scripts](https://github.com/ostreedev/ostree-releng-scripts). 


## Раздельная разаработка vs релизы репозиториев

По умолчанию OSTree накапливает историю на стороне сервера. 
На самом деле это необязательно, поскольку ваша система сборки может (используя API) создать коммит без родителя. 
Но сначала мы исследуем разветвления истории на стороне сервера.

Многие поставщики контента захотят отделить свою внутреннюю разработку от того, что становится общедоступным. 
Следовательно, вам понадобятся (как минимум) два репозитория OSTree, мы назовем их «dev» и «prod».

Другими словами, допустим, у вас есть система непрерывной доставки, которая строится из git и фиксируется в репозитории OSTree для разработчиков. 
Это может происходить от десятков до сотен раз в день. 
Это значительный объем истории с течением времени, и маловероятно, что большинство ваших потребителей контента (т. е. Не разработчиков/тестировщиков) заинтересуются всем этим.

Первоначальное видение OSTree заключалось в том, чтобы выполнять эту роль «разработчика», и, в частности, для этого был разработан формат «archive».

Затем вам нужно будет продвигать контент с «разработчика» на «продакшн». Об этом поговорим позже, но сначала поговорим о «продакшн». otion inside our “dev” repository.


## Продвижение контента по веткам OSTree - «buildmaster», «smoketested»

Помимо нескольких репозиториев, OSTree также поддерживает несколько веток внутри одного репозитория, что эквивалентно ветвям git. 
В предыдущем разделе мы видели пример имени ветки, например exampleos/x86_64/standard. 
Выбор имени ветки для репозитория «prod» крайне важен, поскольку клиентские системы будут ссылаться на него. 
Он становится важной частью вашего лица для мира, точно так же, как «главная» ветвь в репозитории git.

Но с вашим внутренним репозиторием «dev» может быть очень полезно использовать концепции ветвления OSTree для представления различных этапов конвейера доставки программного обеспечения.

Унаследованный от exampleos/x86_64/standard, скажем, наш репозиторий «dev» содержит exampleos/x86_64/buildmaster/standard. 
Мы выбрали термин «buildmaster», чтобы обозначить то, что пришло прямо из git master. Этот контент может не очень гдубоко тестироваться.

Следующим нашим шагом должно быть подключение к нему системы тестирования (Jenkins, Buildbot и т. д.). 
Когда сборка (коммит) проходит несколько тестов, мы хотим «продвинуть» этот коммит. 
Давайте создадим новую ветвь с названием Smoketested, чтобы сказать, что некоторые базовые проверки работоспособности проходят всю систему. 
Здесь могут быть задействованы, например, тестеры.

Данная команда - основной способ «продвинуть» коммит buildmaster, прошедший тестирование:
``
ostree commit -b exampleos/x86_64/smoketested/standard -s 'Passed tests' --tree=ref=aec070645fe53...
``

Здесь мы генерируем новый коммит (возможно, включаем в журнал фиксации ссылки на журналы сборки и т. д.), но мы повторно используем содержимое из фиксации buildmaster aec070645fe53, прошедшей smoketested тесты.

Для более сложной реализации этой модели см. Сценарий do-release-tags, который включает поддержку таких вещей, как распространение номеров версий при продвижении фиксации.

Мы можем легко обобщить эту модель, чтобы иметь произвольное количество этапов, таких как exampleos exampleos/x86_64/stage-1-pass/standard, exampleos/x86_64/stage-2-pass/standard и т. д., в зависимости от бизнес-требований и логики.

В предлагаемой модели «этапы» становятся все более дорогими. 
Логика в том, что мы не хотим тратить много времени, например, на проверка производительности сети, если что-то базовое, например файл модуля systemd, не работает при загрузке. 
## Продвижение контента между репозиториями OSTree

Теперь у нас есть внутренний поток непрерывной доставки (continuous delivery stream flowing), он тестируется и работает. 
Мы хотим периодически брать последний коммит на exampleos/x86_64/stage-3-pass/standard и выставлять его в нашем репозитории «prod» как exampleos/x86_64/standard с гораздо меньшей историей.

У нас будут другие бизнес-требования, такие как написание примечаний к выпуску (и, возможно, их размещение в сообщении о фиксации OSTree) и т. д.

В Build Systems мы увидели, как команду `pull-local` можно использовать для переноса контента из репозитория «build» (в режиме простого пользователя) в архивный репозиторий для передачи в клиентские системы.

После этого раздела у нас теперь есть три репозитория, назовем их repo-build, repo-dev и repo-prod. Мы извлекаем контент из repo-build в repo-dev (который включает, помимо прочего, сжатие gzip, поскольку это изменение формата).

При использовании pull-local для переноса содержимого между двумя архивными репозиториями двоичное содержимое берется без изменений. Давайте продолжим и сгенерируем новый коммит в нашем репозитории prod:
```
checksum=$(ostree --repo=repo-dev rev-parse exampleos/x86_64/stage-3-pass/standard`)
ostree --repo=repo-prod pull-local repo-dev ${checksum}
ostree --repo=repo-prod commit -b exampleos/x86_64/standard \
       -s 'Release 1.2.3' --add-metadata-string=version=1.2.3 \
	   --tree=ref=${checksum}
```

Здесь происходит несколько вещей. Сначала мы нашли последнюю контрольную сумму фиксации для «stage-3 dev» и сказали pull-local скопировать ее без использования имени ветки. 
Мы делаем это, потому что не хотим раскрывать имя ветки exampleos/x86_64/stage-3-pass/standard в нашем репозитории prod.

Затем мы генерируем новую фиксацию в prod, которая ссылается на точное двоичное содержимое в dev. Если репозитории «dev» и «prod» находятся в одной файловой системе Unix, (например, git) OSTree будет использовать жесткие ссылки, чтобы вообще избежать копирования любого контента, что делает процесс очень быстрым.

Еще одна интересная вещь, на которую стоит обратить внимание, - это то, что мы добавляем строку метаданных версии в коммит. Это необязательный элемент метаданных, но мы поощряем его использование в экосистеме инструментов OSTree. Такие команды, как статус администратора ostree, показывают его по умолчанию. 


## Derived data - static deltas and the summary file

As discussed in Formats, the archive repository we use for “prod” requires one HTTP fetch per client request by default. If we’re only performing a release e.g. once a week, it’s appropriate to use “static deltas” to speed up client updates.

So once we’ve used the above command to pull content from repo-dev into repo-prod, let’s generate a delta against the previous commit:
```
ostree --repo=repo-prod static-delta generate exampleos/x86_64/standard
```

We may also want to support client systems upgrading from two commits previous.
```
ostree --repo=repo-prod static-delta generate --from=exampleos/x86_64/standard^^ --to=exampleos/x86_64/standard
```

Generating a full permutation of deltas across all prior versions can get expensive, and there is some support in the OSTree core for static deltas which “recurse” to a parent. This can help create a model where clients download a chain of deltas. Support for this is not fully implemented yet however.

Regardless of whether or not you choose to generate static deltas, you should update the summary file:

ostree --repo=repo-prod summary -u

(Remember, the summary command cannot be run concurrently, so this should be triggered serially by other jobs).

There is some more information on the design of the summary file in Repo.

## Pruning our build and dev repositories

First, the OSTree author believes you should not use OSTree as a “primary content store”. The binaries in an OSTree repository should be derived from a git repository. Your build system should record proper metadata such as the configuration options used to generate the build, and you should be able to rebuild it if necessary. Art assets should be stored in a system that’s designed for that (e.g. Git LFS).

Another way to say this is that five years down the line, we are unlikely to care about retaining the exact binaries from an OS build on Wednesday afternoon three years ago.

We want to save space and prune our “dev” repository.
```
ostree --repo=repo-dev prune --refs-only --keep-younger-than="6 months ago"
```

That will truncate the history older than 6 months. Deleted commits will have “tombstone markers” added so that you know they were explicitly deleted, but all content in them (that is not referenced by a still retained commit) will be garbage collected.

## Generating “scratch” deltas for efficient initial downloads

In general, the happy path for OSTree downloads is via static deltas. If you are in a situation where you want to download an OSTree commit from an uninitialized repo (or one with unrelated history), you can generate “scratch” (aka --empty deltas) which bundle all objects for that commit.

The tradeoff here is increasing server disk space in return for many fewer client HTTP requests.

For example:
```
$ ostree --repo=/path/to/repo static-delta generate --empty --to=exampleos/x86_64/testing-devel
$ ostree --repo=/path/to/repo summary -u
```

After that, clients fetching that commit will prefer fetching the “scratch” delta if they don’t have the original ref.
