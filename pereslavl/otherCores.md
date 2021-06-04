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

Дистрибутив | Fedora CoreOS | Fedora IoT | Flatcar Container Linux | Ubuntu Core | Chrome OS | GNOME OS
------------|---------------|------------|-------------------------|-------------|-----------|----------
Пакетная база | Fedora      | Fedora     | Gentoo                  | Ubuntu 
Потоки(Streams) | Next, Testing, Stable |  Stable | Alpha, Beta, Stable, LTS | Releases | 
Платформа   | x86_64 | x86_64, aarch64, armhfp | amd64. arm64      | amd64. arm64
Ниша | Cloud, VM, BM        |  ED         |   Cloud, VM, BM        | ED
ПО мультидеплоя | ostree    | ostree      |                        | snap
Пакетный менеджер | rpm-ostree | rpm-ostree | -                    | Snappy 
Ignition    |  Да           | Да          | Да                     | Нет
Атомарность | развертывания | развертывания | развертывания        | Ядро?
Автообновление | Да         | Нет         | Да                     | 
Хранение развертываний | H  | H           | S (USR-A, USR-B)       |      
Откат(rollback) | Да         | Да         | Да                     | только ядро
ReadOnly дерево | /usr      | /usr        | /usr
Шифрование диска |  Да      |  Да         | ?                      | Да
Системные сервисы | systemd, sssd , zincati | systemd, parces, zezere | systemd, etcd, 

Системные сервисы
- sssd - [System Security Services Daemon](https://en.wikipedia.org/wiki/System_Security_Services_Daemon)
- zincati - [Zincati](https://github.com/coreos/zincati) is an auto-update agent for Fedora CoreOS hosts.
- parces - [PARSEC](https://fedoraproject.org/wiki/Changes/PARSEC) (Platform AbstRaction for SECurity) is an initiative started out of Arm to provide a straight forward API for accessing secure credentials stored in hardware. It's a sandbox project in the CNCF. 
- Zezere -  [Zezere](https://github.com/fedora-iot/zezere) is a provisioning server for Fedora IoT. It can be used for deploying Fedora IoT to devices without needing a physical console
- etcd - [etcd](https://etcd.io/) - A distributed, reliable key-value store for the most critical data of a distributed system

Нишы:
- Cloud - мультиоблачный кластер
- VM - вирруальные машины
- BM - голое железо (Bare Metal)
- ED - встраиваемые устройства (Embedded devices )

Хранение развертываний:
- H - Деревья развертываний залинкованных (HardLink) на общую базу файлов-объектов
- S - Отдельные (Separate) деревья

### Flatcar Container Linu

- [Документация](https://kinvolk.io/docs/flatcar-container-linux/latest/)
- [Flatcar Container Linux disk layout](https://kinvolk.io/docs/flatcar-container-linux/latest/reference/developer-guides/sdk-disk-partitions/)


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
