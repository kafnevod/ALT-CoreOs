# Другие Ciore

- [Ububtu Core](https://ubuntu.com/core)
  * [What is Ubuntu Core?](https://ubuntu.com/core/docs/what-is-ubuntu-core)
  * [Image building](https://ubuntu.com/core/docs/image-building)
- [CoreOS](https://ru.wikipedia.org/wiki/CoreOS)
- [Flatcar Container Linux](https://kinvolk.io/flatcar-container-linux/) (CoreOS Container Linux)
- [Ubuntu ImageBasedUpgrades](https://wiki.ubuntu.com/ImageBasedUpgrades)
- [Chrome OS](https://ru.wikipedia.org/wiki/Chrome_OS)
- [GNOME OS](https://wiki.gnome.org/action/show//GnomeOS?action=show&redirect=Projects%2FGnomeContinuous)

## Обзор

Дистрибутив | Fedora CoreOS | Fedora IoT | Flatcar Container Linux | Ubuntu Core 
------------|---------------|------------|-------------------------|-------------
Пакетная база | Fedora      | Fedora     | Gentoo                  | Ubuntu 
Потоки(Streams) | Next, Testing, Stable |  Stable | Alpha, Beta, Stable, LTS
Платформа   | x86_64 | x86_64, aarch64, armhfp | amd64. arm64      | amd64. arm64
Ниша | Cloud, VM, BM        |  ED         |   Cloud, VM, BM        | ED
ПО мультидеплоя | ostree    | ostree      |                        | snap
Пакетный менеджер | rpm-ostree | rpm-ostree | -                    | Snappy 
Ignition    |  Да           | Да          | Да                     | Нет
Атомарность | развертывания | развертывания | развертывания        | Ядро?
Автообновление | Да         | Да          | Да                     | 
Хранение развертываний | H  | H           | S 
Откат(rollback) | Да         | Да          |                       | только ядро
ReadOnly дерево | /usr      | /usr        |
Шифрование диска |  Да      |  Да         |                        | Да


Нишы:
- Cloud - мультиоблачный кластер
- VM - вирруальные машины
- BM - голое железо (Bare Metal)
- ED - встраиваемые устройства (Embedded devices )

Хранение развертываний:
- H - Деревья rfnfkjujd залинкованных (HardLink) на общую базу файлов-объектов
- S - Отдельные (Separate) деревья

### Ubuntu Core

- ориентировано тольео на IoT (нет stansalone)
- основное ПО ставится как snap-пакеты (ReadOnly FS)
- генерация образа через [Ubuntu image builder](https://github.com/CanonicalLtd/ubuntu-image)
- заназные (bespoke) образы производится хточно также (https://ubuntu.com/core/docs/custom-images) 
  *  Model assertion
  *  Prerequisites
  *  Custom model assertion
  *  Signing a model assertion
  *  Building the image

### CoreOS (Container Linux)
- Fork Chromium OS
- ориентирована на севреные облачные плаиформы
- нет пакетного менеджера - все ПО в контейнерах
- две фйловые системе - рабочая RO и  следующая RWпсоле установки ПО меняются местами

Судя по ссылкам полностью поглощена RedHat
