# libostree

    - Операционные системы и дистрибутивы, использующие OSTree 
    - Инструменты сборки дистрибутива
    - Проекты, связанные с libostree
    - Language bindings

Этот проект теперь известен как «`libostree`», хотя по-прежнему уместно использовать предыдущее название: «`OSTree`» (или «`ostree`»).  Однако в большей части остальной документации мы будем использовать термин «`OSTree`», поскольку он немного короче, и изменять сразу всю документацию нецелесообразно. Мы ожидаем перехода на новое имя со временем. 

В данном документе основное внимание уделяется проектам, которые используют общую библиотеку `libostree`, а не пользователям, напрямую вызывающим инструменты командной строки (за исключением систем сборки).

Как упоминалось выше, `libostree` - это как совместно используемая библиотека, так и набор инструментов командной строки, которые объединяют «`git-подобную`» модель для фиксации и загрузки деревьев загрузочных файловых систем, а также уровень их развертывания и управления конфигурацией загрузчика.

Базовая модель `OSTree` похожа на `git` тем, что в ней проверяются суммы отдельных файлов и есть хранилище объектов с адресацией к содержимому. В отличие от `git`, она «проверяет» файлы с помощью жестких ссылок, и поэтому они должны быть неизменными, чтобы предотвратить повреждение. Таким образом  `OSTree` можно представить как  более совершенную версию жестких ссылок (`hard links`) `Linux VServer`. 

**Особенности:**

- Транзакционные обновления и откат для операционной системы.
- Инкрементальная репликация контента через `HTTP` с помощью подписей `GPG` и поддержки «прикрепленного (`pinned`) `TLS`».
- Поддержка параллельной установки двух и более загрузочных корневых файловых систем.
- Бинарная история на стороне сервера (и клиента).
- Общий `API` для систем сборки и развертывания
- Гибкая поддержка нескольких веток и репозиториев.


## Операционные системы и дистрибутивы, использующие OSTree 

[Endless OS](https://endlessos.com/) использует `libostree` для своей хост-системы, а также `flatpak`. 
См. [eхos-updater](https://github.com/endlessm/eos-updater)
и [deb-ostree-builder](https://github.com/dbnicholson/deb-ostree-builder). 

Три подпроекта Fedora используют rpmostree: 
- [Fedora CoreOS](https://getfedora.org/en/coreos/)
- [Fedora Silverblue](https://silverblue.fedoraproject.org/)
- [Fedora IoT](https://getfedora.org/iot/)

`Red Hat Enterprise Linux CoreOS` - это производная от `Fedora CoreOS`, используемая в [OpenShift 4](https://www.openshift.com/try). 
[machine-config-operator](https://github.com/openshift/machine-config-operator/blob/master/docs/OSUpgrades.md) управляет обновлениями. 
`RHEL CoreOS` также является преемником `RHEL Atomic Host`, который также использует `rpm-ostree`. 

[GNOME Continuous](https://wiki.gnome.org/action/show//GnomeOS?action=show&redirect=Projects%2FGnomeContinuous) - это то место, 
где возник проект `OSTree` - как высокопроизводительная система `continuous delivery/testing` для `GNOME`. 

[Liri OS](https://liri.io/download/silverblue/) под `Fedora Silverblue` поддерживает возможность установить свой дистрибутив с помощью `ostree`. 

[TorizonCore](https://developer.toradex.com/knowledge-base/torizoncore-overview) использует `libostree` и 
`Aktualizr` в качестве основы для обновлений `OTA` совместимых платформ, 
включая [Torizon OTA](https://developer.toradex.com/knowledge-base/torizon-update-system). 

## Инструменты сборки дистрибутива

[meta-updater](https://github.com/advancedtelematic/meta-updater) - это слой, доступный для [OpenEmbedded](http://www.openembedded.org/wiki/Main_Page)-систем.

[QtOTA](https://doc.qt.io/archives/QtOTA/) - это среда беспроводного обновления `Qt`, использующая `libostree`.

Инструмент сборки и интеграции [BuildStream](https://github.com/apache/buildstream/) поддерживает импорт и экспорт из репозиториев `libostree`.

`Fedora` [coreos-assemblyr](https://github.com/coreos/coreos-assembler) - это инструмент сборки, используемый для создания производных от `Fedora CoreOS`. 

## Проекты, связанные с libostree

[rpm-ostree](https://github.com/coreos/rpm-ostree) используется перечисленными выше операционными системами, производными от `Fedora`. Это полная гибридная система образов/пакетов. По умолчанию он использует `libostree` для атомарной репликации базовой ОС (все разрешение зависимостей выполняется на сервере), но он поддерживает «многоуровневый пакет», при котором дополнительные `RPM пакеты` в виде отдельного слоя 
могут быть размещены поверх базовой, совмещая преемущества обоих подходов.

[eos-updater](https://github.com/endlessm/eos-updater) - это демон, который выполняет обновления в `EndlessOS`.

[Flatpak](https://github.com/flatpak/flatpak) использует `libostree` для контейнеров настольных приложений. 
В отличие от большинства других систем здесь, `flatpak` не использует аспекты `«хост-системы libostree»` (например, управление загрузчиком), а только «дедупликацию жестких ссылок, подобную `git`». 
Например, `flatpak` поддерживает репозиторий `OSTree` для каждого пользователя. 

## Language bindings

libostree доступен через [GObject Introspection](https://gi.readthedocs.io/en/latest/). 
Любой язык, в котором реализована `GI binding model`, должен работать. Например, известно, что как `pygobject`, так и `gjs` работают, и сегодня они фактически используются в наборе тестов `libostree`.

Некоторые `bindings` используют `GI` в качестве нижнего уровня и записывают ручные `bindings` более высокого уровня сверху; это более характерно для статически компилируемых языков. Ниже список  привязок:
- `ostree-go`
- `ostree-rs`


