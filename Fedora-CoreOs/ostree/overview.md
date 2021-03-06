# Обзор OSTree 

-    Введение
-    Пример Hello World
-    Сравнение с «менеджерами пакетов» rpm, dpkg, ...
-    Сравнение с репликацией блоков/образов (image based)

## Введение

`OSTree` - это система обновления для операционных систем на базе `Linux`, которая выполняет атомарное обновление полных деревьев файловых систем. 
Это не одна из пакетных систем; скорее, она предназначена для их дополнения. 
Основная модель - это составление пакетов на сервере, а затем их репликация  клиентам.

Базовую архитектуру можно кратко охарактеризовать как «`git для двоичных файлов операционной системы`». 
Она работает в пользовательском пространстве и будет работать поверх любой файловой системы `Linux`. 
По своей сути, это хранилище объектов с адресацией к содержимому, похожее на `git`, с ветвями (`branches`) (или «`refs`») для отслеживания значимых деревьев файловых систем в хранилище. 
Точно так же можно проверить или закоммитить (`commit`) эти ветки.

На вершине этого уровня находится конфигурация загрузчика, управление каталогом `/etc` и т.д., а также помимо репликации файлов, другие функции для выполнения обновления.

Вы можете использовать OSTree автономно в чистой модели репликации, но другой подход заключается в добавлении диспетчера пакетов поверх, создавая таким образом гибридную систему `корневого дерева файловой системы`/`пакетов`. 


## Пример Hello World

`OSTree` в основном используется как библиотека, но краткий обзор использования его инструментов `CLI `может дать общее представление о том, как он работает на самом базовом уровне.

Вы можете создать новый репозиторий `OSTree` с помощью `init`: 
```
$ ostree --repo=repo init
```

Это создаст новый каталог `repo`, содержащий ваш репозиторий. 
Теперь давайте подготовим данные для добавления в репо: 
```
$ mkdir tree
$ echo "Hello world!" > tree/hello.txt
```
Теперь мы можем импортировать наше каталог `tree` с помощью команды `commit`: 
```
$ ostree --repo=repo commit --branch=foo tree/
```
Это создаст новую ветку `foo`, указывающую на полное дерево, импортированное из `tree/`. Фактически, теперь мы могли удалить `tree/`, если бы захотели.

Чтобы убедиться, что у нас действительно есть ветка foo, вы можете использовать команду `refs`: 
```
$ ostree --repo=repo refs
foo
```

Мы также можем просмотреть дерево файловой системы с помощью команд `ls` и `cat`: 
```
$ ostree --repo=repo ls foo
d00775 1000 1000      0 /
-00664 1000 1000     13 /hello.txt
$ ostree --repo=repo cat foo /hello.txt
Hello world!
```

И, наконец, мы можем выполнить `checkout`  нашего дерево из репозитория: 
```
$ ostree --repo=repo checkout foo tree-checkout/
$ cat tree-checkout/hello.txt
Hello world!
```

## Сравнение с «менеджерами пакетов» rpm, dpkg, ...

Поскольку `OSTree` разработан для развертывания основных операционных систем, сравнение с традиционными «менеджерами пакетов», такими как `dpkg` и `rpm`, является показательным. Пакеты традиционно состоят из частичных деревьев файловых систем с прикрепленными метаданными и скриптами, которые динамически собираются на клиентском компьютере после процесса разрешения зависимостей.

Напротив, `OSTree` поддерживает только запись и развертывание полных (загрузочных) деревьев файловых систем. 
Он не имеет встроенных знаний о том, как было сгенерировано данное дерево файловой системы, или о происхождении отдельных файлов, или о зависимостях, и о описаниях отдельных компонентов. 
Другими словами, `OSTree` занимается только доставкой и развертыванием.  
Вы, вероятно, по-прежнему захотите включить в каждое дерево метаданные об отдельных компонентах, которые вошли в дерево. Например, системный администратор может захотеть узнать, какая версия `OpenSSL` была включена в ваше дерево, поэтому вы должны поддерживать эквивалент 
`rpm -q` (*в `Fedora CoreOs` это rpm-ostree*) или `dpkg -L`.

Ядро `OSTree` делает упор на репликацию деревьев ОС, доступных только для чтения, через `HTTP`, и где ОС включает (при желании) полностью отдельный механизм для установки приложений, хранящихся в `/var`, если они являются глобальными для системы, или `/home` для установки приложений для каждого пользователя. 
Пример механизма приложения - http://docker.io/

Однако вполне возможно использовать `OSTree` под системой пакетов, где содержимое `/usr` вычисляется на клиенте. 
Например, при установке пакета вместо изменения текущей файловой системы диспетчер пакетов может собрать новое дерево файловой системы, 
которое накладывает новые пакеты поверх базового дерева, записать его в локальный репозиторий `OSTree`, а затем настроить его для следующей загрузки 
(*реализовано в rpm-ostree Fedora Core*). 
Для поддержки этой модели `OSTree` предоставляет (интроспективную) разделяемую библиотеку `C.` 

## Сравнение с репликацией блоков/образов (image based)

`OSTree` имеет некоторое сходство с «прозрачной» репликацией и развертываниями без сохранения состояния, например с моделью, распространенной в «облачных» развертываниях, где узлы загружаются с диска (фактически) только для чтения, а пользовательские данные хранятся на разных томах. Преимущество такой репликации, реализованной как в `OSTree`, так и в `облачной модели`, состоит в том, что она надежна и предсказуема.

Но в отличие от многих `image based` развертываний,  `OSTree` поддерживает ровно два постоянных каталога с возможностью записи, которые сохраняются при обновлении: `/etc` и `/var`.

Поскольку `OSTree` работает на уровне файловой системы `Unix`, он работает поверх любой файловой системы или структуры блочного хранилища; можно реплицировать заданное дерево файловой системы из репозитория `OSTree` в обычный **`ext4`**, BTRFS, XFS (*реализовано в Fedora Core Os*) или вообще любую `Unix-совместимую `файловую систему, которая поддерживает `жесткие` (`hard`) ссылки. 

> Примечание. Для файловой системы `BTRFS` `OSTree` поддерживает дополнительные функции, реализованные в `BTRFS`.

`OSTree` ортогонален механизмам виртуализации на основе образов виртуальных машин типа `AMI` и `qcow2`.
Его преимущаство в том, что он позволает настраивать развернутые на узлах стандартные образы сгенерированные для каждого узла.
не загружая полностью `AMI` или `qcow2` сгенерированные образы.    

На практике для пользователей конфигураций с «`голым железом»` (`metal bare`) наиболее полезной будет модель `OSTree`. 

## Атомарные переходы между устанавливаемыми параллельно деревьями файловых систем только для чтения

Еще одно фундаментальное различие между менеджерами пакетов и репликацией на основе образов состоит в том, что `OSTree` предназначен для параллельной установки нескольких версий нескольких независимых операционных систем. 
`OSTree` использует отдельный каталог `ostree` верхнего уровня; фактически он может быть установлен параллельно внутри существующей ОС или дистрибутива, занимающего физический `/ ` - корневой каталог.

На каждой клиентской машине есть репозиторий `OSTree`, хранящийся в `/ostree/repo`, и множество «развертываний», хранящийся в `/ostree/deploy/$STATEROOT/$CHECKSUM`. Каждое развертывание (`deploy`) в основном состоит из набора жестких ссылок на репозиторий. Это означает, что каждая версия дедуплицирована (одинаковые файлы разных развертываний хранятся в единичном экземпляре); процесс обновления требует только дискового пространства, пропорционального количеству новых файлов, плюс постоянные накладные расходы.

Модель `OSTree` требует, чтобы содержимое ОС, доступное только для чтения, хранится в классическом `Unix`  каталоге `/usr`.
Он включает с код обеспечивающий 
`read-only bind mount` каталога `/usr`, чтобы предотвратить случайное повреждение. 
Существует **ровно один** доступный для записи каталог `/var`, совместно используемый каждым развертыванием данной ОС. 
Основной код `OSTree` не касается содержимого в этом каталоге. 
То, как управлять состоянием и обновлять его, зависит от кода каждой операционной системы.

Наконец, каждое развертывание имеет свою собственную доступную для записи копию хранилища конфигурации `/etc`. 
При обновлении `OSTree` выполнит базовое трехстороннее сравнение (`3-way diff`) и применит любые локальные изменения к новой копии, 
оставляя старую нетронутой. 
