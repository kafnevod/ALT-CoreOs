# Другие Ciore

- [Ububtu Core](https://ubuntu.com/core)
  * [What is Ubuntu Core?](https://ubuntu.com/core/docs/what-is-ubuntu-core)
  * [Image building](https://ubuntu.com/core/docs/image-building)
- [CoreOS](https://ru.wikipedia.org/wiki/CoreOS)
- [Flatcar Container Linux](https://kinvolk.io/flatcar-container-linux/)

## Обзор

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

### 

Дистрибутив | Дистр | Платформы | Ниша | ПО мультидеплоя | Ignition | Атомарность на уровне | Автообновление |Хранение развертываний | Откат | ReadOnly дерево | Обновление /etc  
------------|-------|-----------|-----------|------------|----------|-----------------------|----------------|-----------------------|----------|-----------------|-----------------
Fedora CoreOS | Frdora |  | мультиоблачный кластер | ostree | Да | развертывания | Да | Деревья залинкованных на общую базу файлов | Да | /usr | трехстороноее слияние
Fedora IoT | Frdora |  | Интернет вещей | ostree | Да| развертывания |  Да | Деревья залинкованных на общую базу файлов | Да | /usr | трехстороноее слияние
[Flatcar Container Linux](https://kinvolk.io/flatcar-container-linux/) (форк CoreOS Containet Linux)| Gentoo | мультиоблачный кластер | | | Да | | | | | | |
[Ubuntu Core](https://ubuntu.com/core) |Ubuntu | Интернет вещей | | snapd | | пакета |  | | | | |
[Ubuntu ImageBasedUpgrades](https://wiki.ubuntu.com/ImageBasedUpgrades) | Ubuntu | | | | | | | | |
[NixOS](https://nixos.org/guides/how-nix-works.html)|  | | | | | | | | |
[Chrome OS](https://ru.wikipedia.org/wiki/Chrome_OS)|  | | | | | | | | |
[GNOME OS](https://wiki.gnome.org/action/show//GnomeOS?action=show&redirect=Projects%2FGnomeContinuous)|  | | | | | | | | | 

Дистрибутив | Fedora CoreOS | Fedora IoT | Flatcar Container Linux | Ubuntu Core 
------------|---------------|------------|-------------------------|-------------
Пакетная база | Fedora      | Fedora     | Gentoo                  | Ububtu 
Платформа   | x86_64 | x86_64, aarch64, armhfp |                   |
Ниша | Cloud, VM, BM        |  E          |                        |
ПО мультидеплоя | ostree    | ostree      |                        |
Ignition    |  Да           | Да          | Да                     | 
Атомарность | развертывания | развертывания | развертывания        | 
Автообновление | Да         | Да          | Да                     | 
Хранение развертываний | H  | H           | S 
Откат(rollback) | Да         | Да          | 
ReadOnly дерево | /usr      | /usr        |

Нишы:
- Cloud - мультиоблачный кластер
- VM - вирруальные машины
- BM - голое железо (Bare Metal)
- E - встраиваемые устройства (Embedded devices )

Хранение развертываний:
- H - Деревья rfnfkjujd залинкованных (HardLink) на общую базу файлов-объектов
- S - Отдельные (Separate) деревья

