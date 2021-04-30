# Описание дистрибутива Fedore CoreOs

Отличительными особенностями дистрибутива Fedore CoreOs являются:
- отсутствие этапа "ручной" настройки. Все нижеописанные настройки задаются в файле в формате ignition передаваемому параметром команде формирования загрузочного образа дистрибутива `coreos-installer` 
- использование `ostree` для обеспечения "атомарного" обновления дистрибутива. При добавлении удалении пакетов командой `rpm-ostree` формируется новый слой ПО, который становится доступном только после перезагрузки ОС.
- четкая классификация корневых директориев:
  * `/usr/` - неизменяемая в рамках текущей загрузки часть программного кода (пакетов), обычно монтируется в режиме `только на чтение`
  * `/etc/` - конфигурационная часть программного кода, во время обновления пакетов либо не модифицируется, либо донастраивается скриптами пакета;'
  * `/var/` - изменяемая часть - логи, домашние директории пользователей, обрабатываемые данные;
  * остальные корневые директории либо носят служебный (/dev/) и временный характер (/tmp), либо символически прилинкованы к поддиректориям перечисленным выше жиректориев.
  


## Установка дистрибутива

### Ignition

Файл формата ignition (JSON-файл с суффиксом .ign) обеспечивает:
 - разбиение дисков на партиции (storag.disks.device.patrition, storage.filesystems);
 - создание защифрованных носителей (boot_device.luks) данные в формате [LUKS](https://coreos.github.io/butane/examples/#luks-encrypted-storage);
 - создание [программных RAID-массивов](https://coreos.github.io/butane/examples/#mirrored-boot-disk) (boot_device.mirror);
 - создание пользователей (passwd.users) и групп (passwd.groups);
 - запуск и конфигурирование сервисов (systems.units) включая docker-контейнеры;
 - добавление или замена директориев (storage.directories) и(конфигурационных) файлов (storage.files, storage.links), что позволяет сконфигурировать:
    * сетевые интерфейсы (storage.files.path=/etc/NetworkManager/system-connections/...);  
    * параметры ядра (rpm-ostree kargs ..., storage.files.path=/etc/sysctl.d/...).
    * ...
   
Файл имеет формат JSON и формируется обычно двумя способами:
- программой конфигурации;
- программой [butane](https://coreos.github.io/butane/) на основе созданного в пользовательском редакторе или автоматически [файла формата YML](https://coreos.github.io/butane/examples/). 

### coreos-installer

[Документация](https://coreos.github.io/coreos-installer/)

Платформа | Форматы 
-----------|----------
Bare Metal | iso, pxe,  4k.raw.xz, row.xz
Alibaba Cloud (Aliyun) | qcow2.xz
Amazon Web Services | Напрямую с https://builds.coreos.fedoraproject.org/
Azure | vhd.xz
DigitalOcean | Напрямую с https://builds.coreos.fedoraproject.org/
Exoscale| qcow2.xz, напрямую с https://builds.coreos.fedoraproject.org/
Google Cloud Platform (GCP) | Напрямую с https://builds.coreos.fedoraproject.org/
IBM Cloud | qcow2.xz
libvirt | qcow2.xz
OpenStack | qcow2.xz
QEMU | qcow2.xz
VMware | ova
Vultr | raw.xz


### Структура LIVE PXE и ISO образов

- fedora-coreos-33.20210328.3.0-live-kernel-x86_64:        Linux kernel x86 boot executable RO-rootFS, swap_dev 0xA, Normal VGA
- fedora-coreos-33.20210328.3.0-live-initramfs.x86_64.img:
```
drwxrwxr-x   3 root     root            0 Jan  1  1970 .
-rw-rw-r--   1 root     root            2 Jan  1  1970 early_cpio
drwxrwxr-x   3 root     root            0 Jan  1  1970 kernel
drwxrwxr-x   3 root     root            0 Jan  1  1970 kernel/x86
drwxrwxr-x   2 root     root            0 Jan  1  1970 kernel/x86/microcode
-rw-rw-r--   1 root     root        30546 Jan  1  1970 kernel/x86/microcode/AuthenticAMD.bin
-rw-rw-r--   1 root     root      3623936 Jan  1  1970 kernel/x86/microcode/GenuineIntel.bin
```
- fedora-coreos-33.20210328.3.0-live-rootfs.x86_64.img:
```
-rw-r--r--   1 root     root     696107008 Apr 12 20:52 root.squashfs
drwxr-sr-x   2 root     root            0 Apr 12 20:43 etc
-rw-r--r--   1 root     root           16 Apr 12 20:43 etc/coreos-live-rootfs
-rw-------   1 root     root      5701465 Apr 12 20:44 fedora-coreos-33.20210328.3.0-metal.x86_64.raw.osmet
-rw-------   1 root     root      5507327 Apr 12 20:46 fedora-coreos-33.20210328.3.0-metal4k.x86_64.raw.osmet
```
  * root.squashfs:
```
.coreos-aleph-version.json
boot
    /boot -> .
    /bootupd-state.json
    /efi/...
    /grub2/...
    /ignition.firstboot
    /loader-> loader.1
    /loader.1/entries/ostree-1-fedora-coreos.conf
    /lost+found/
    /ostree/fedora-coreos-7bbc78c7f5c49e3bd679517942d74bf1c3024647716b30a01aae83bb4102244b/
        initramfs-5.10.19-200.fc33.x86_64.img
        vmlinuz-5.10.19-200.fc33.x86_64
ostree
      /boot.1 -> boot.1.1
      /boot.1.1/fedora-coreos/7bbc78c7f5c49e3bd679517942d74bf1c3024647716b30a01aae83bb4102244b/0/
          boot/
          dev/
          etc/
          ...
          usr/
          var/
      /deploy/fedora-coreos/
          deploy/0fc4a4c205dbcdfd6ba68912bfbf2c90911e4a2341b3dda0a254ab6541224b83.0
              boot/
              dev/
              etc/
              ...
              usr/
          var/
              lib/
              log/
              tmp/
      /repo
          config
          extensions
          objects
          refs
          state
          tmp 
```

#### Замечания по поводу файлов формата osmet (ostree + metal)

Fedora  CoreOS поддерживает загрузку в реальных средах PXE и ISO. Они работают с использованием «rootfs initrd», который содержит squashfs реальных rootfs для загрузки. Этот rootfs, как и любая система на основе OSTree, содержит системное репозиторий OSTree, из которого объекты жестко связаны для заполнения дерева корневой файловой системы.

На практике основным вариантом использования этих сред является простая установка CoreOS на диск и перезагрузка установленной системы. В прошлом установка означала получение raw-образа металла из удаленного репозитория и запись его на диск. Однако это неэффективно, потому что большая часть данных на этом изображении поступает из тех же объектов OSTree, которые уже присутствуют в squashfs.

Функциональность osmet - позволяет coreos-installer повторно использовать эти объекты для установки на диск, в то же время побитово сопоставляя содержимое metal-образа. 

Подробности [здесь](https://coreos.github.io/coreos-installer/osmet/).

