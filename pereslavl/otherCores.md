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

### CoreOS (Cmtainer Linux
- Fork Chromium OS
- ориентирована на севреные облачные плаиформы
- нет пакетного менеджера - все ПО в контейнерах
- две фйловые системе - рабочая RO и  следующая RWпсоле установки ПО меняются местами

Судя по ссылкам полностью поглощена RedHat

### 

Дистрибутив | Ниша | Платформы | ПО мультидеплоя | Атомарность на уровне | Автообновление |Хранение развертываний | Откат | ReadOnly дерево | Обновление /etc  
------------|------|-----------|-----------------|-----------------------|----------------|-----------------------|----------|-----------------|-----------------
Fedora CoreOS |  |мультиоблачный кластер | ostree | развертывания | Да | Деревья залинкованных на общую базу файлов | Да | /usr | трехстороноее слияние
Fedora IoT |  | Интернет вещей | ostree | развертывания |  Да | Деревья залинкованных на общую базу файлов | Да | /usr | трехстороноее слияние
[Flatcar Container Linux](https://kinvolk.io/flatcar-container-linux/) (CoreOS Containet Linux)| | | | | | | | |
[Ububtu Core](https://ubuntu.com/core) | | Интернет вещей | snapd | пакета | | | | |
[Ubuntu ImageBasedUpgrades](https://wiki.ubuntu.com/ImageBasedUpgrades) | | | | | | | | |
[NixOS](https://nixos.org/guides/how-nix-works.html) | | | | | | | | |
[Chrome OS](https://ru.wikipedia.org/wiki/Chrome_OS) | | | | | | | | |
[GNOME OS](https://wiki.gnome.org/action/show//GnomeOS?action=show&redirect=Projects%2FGnomeContinuous) | | | | | | | | | 

