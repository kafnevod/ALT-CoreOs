# Описание процедуры загрузки образа Fedora CoreOs для различных потоков

Загрузку и установку образа производит команда [coreos-installer](https://github.com/coreos/coreos-installer).

В зависимости от указанного потока (next, testing, stable) `coreos-installer` заправшивает мета-информацию по URL:
`https://builds.coreos.fedoraproject.org/streams/<имя_потока>.json`.

## Структура метаданных описания образов потока

Рассмотрим структуру метаданных описания образов потока `stable`.
Она расположена по URL: `https://builds.coreos.fedoraproject.org/streams/stable.json`.
Верхний уровень JSON-описания выглядит следующим образом:
```
{
    "stream": "stable",
    "metadata": {
        "last-modified": "2021-06-15T13:01:11Z"
    },
    "architectures": {
      ...
    }
}
```
### Описание архитекутр (architectures)

```
    "architectures": {
        "x86_64": {
            "artifacts": {
                "aliyun": { ... },
                "aws": { ... },
                "azure": { ... },
                "digitalocean": { ... },
                "exoscale": { ... },
                "gcp": { ... },
                "ibmcloud": { ... },
                "metal": { ... },
                "openstack": { ... },
                "qemu": { ... },
                "vmware": { ... },
                "vultr": { ... }
            },
            "images": {
                "aws": {
                    "regions": {
                        "af-south-1": {
                            "release": "34.20210529.3.0",
                            "image": "ami-0d9658cc46b2bba75"
                        },
                        "ap-east-1": {
                            "release": "34.20210529.3.0",
                            "image": "ami-0af9744b58d7daa93"
                        },
                        ...
                    }
                },
                "gcp": {
                    "project": "fedora-coreos-cloud",
                    "family": "fedora-coreos-stable",
                    "name": "fedora-coreos-34-20210529-3-0-gcp-x86-64"
                }                }
            }
        }
}
```
В настоящий момент (`24.06.2021`) поддерживается только архитектура `x86_64`.
Данная архитектура содержит два подэлемента:
- `arfifacts` содержащий описания образов для различный `облачных платформ`, `голого железа` и `qemy`.
- `images` - содержащий дополнитаельныю информацию по облачным платформам `aws` и `gcp`.

### Описние платформ `artifacts`

Описание платформы формируется по шаблону:
```
        {
            "release": "34.20210529.3.0",
            "formats": {
            "<формат>": {
                "disk": {
                    "location": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/...",
                    "signature": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/...sig",
                    "sha256": "...",
                    "uncompressed-sha256": "..."
                }
            },
        }
```
В случае, если образ не сжат поле `uncompressed-sha256` отсутсвует.

Форматы образов для различных сред:
Среда развертывания | iso | ova | pxe | qcow2.xz | raw.xz | 4k.raw.xz | tar.gz | vhd.xz | vmdk.xz |
--------------------|-----|-----|-----|----------|--------|-----------|--------|--------|---------|
aliyun              |     |     |     |     V    |        |           |        |        |         |
aws                 |     |     |     |          |        |           |        |        |    V    |
azure               |     |     |     |          |        |           |        |    V   |         |
digitalocean        |     |     |     |      V   |        |           |        |        |         |
exoscale            |     |     |     |      V   |        |           |        |        |         |
gcp                 |     |     |     |          |        |           |   V    |        |         |
ibmcloud            |     |     |     |      V   |        |           |        |        |         |
metal               |   V |     |   V |          |    V   |     V     |        |        |         |
openstack           |     |     |     |      V   |        |           |        |        |         |
qemu                |     |     |     |      V   |        |           |        |        |         |
vmware              |     |   V |     |          |        |           |        |        |         |
vultr               |     |     |     |          |    V   |           |        |        |         |

Формат описания среды `pxe` отличается от описанного выше и имеет вид:
```
            "kernel": {
                "location": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/...",
               "signature": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/....sig",
                "sha256": "..."
            },
            "initramfs": {
                "location": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/...img",
                "signature": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/....img.sig",
                "sha256": "..."
            },
            "rootfs": {
                "location": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/...img",
                "signature": "https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/34.20210529.3.0/x86_64/...img.sig",
                "sha256": "..."
            }
```

### Структура ISO-образа для среды `голого железа`

ISO-образ имеет следующую структуру:
```
# ls -lR 
.:
итого 7
dr-xr-xr-x 3 root root 2048 июн 14 20:26 EFI
dr-xr-xr-x 3 root root 2048 июн 14 20:26 images
dr-xr-xr-x 2 root root 2048 июн 14 20:26 isolinux
-r--r--r-- 1 root root  114 июн 14 19:13 zipl.prm

./EFI:
итого 2
dr-xr-xr-x 2 root root 2048 июн 14 20:26 fedora

./EFI/fedora:
итого 3
-r--r--r-- 1 root root 2119 июн 14 19:13 grub.cfg

./images:
итого 7294
-r--r--r-- 1 root root 7204454 июн 14 20:26 efiboot.img
-r--r--r-- 1 root root  262144 июн 14 20:18 ignition.img
dr-xr-xr-x 2 root root    2048 июн 14 20:24 pxeboot

./images/pxeboot:
итого 718665
-r--r--r-- 1 root root  73907404 июн 14 20:26 initrd.img
-r--r--r-- 1 root root 651373056 июн 14 20:24 rootfs.img
-r--r--r-- 1 root root  10631888 июн 14 20:18 vmlinuz

./isolinux:
итого 383
-r--r--r-- 1 root root   2048 июн 14 20:26 boot.cat
-r--r--r-- 1 root root     58 июн 14 19:13 boot.msg
-r-xr-xr-x 1 root root  38912 июн 14 20:26 isolinux.bin
-r--r--r-- 1 root root   3081 июн 14 19:13 isolinux.cfg
-r-xr-xr-x 1 root root 115844 июн 14 20:26 ldlinux.c32
-r-xr-xr-x 1 root root 179456 июн 14 20:26 libcom32.c32
-r-xr-xr-x 1 root root  23508 июн 14 20:26 libutil.c32
-r-xr-xr-x 1 root root  26696 июн 14 20:26 vesamenu.c32
```

Содержание файла ``EFI/fedora/grub.cfg:
```
# Note this file mostly matches the grub.cfg file from within the
# efiboot.img on the Fedora Server DVD iso. Diff this file with that
# file in the future to pick up changes.
#
# One diff to note is we use linux and initrd instead of linuxefi and
# initrdefi. We do this because it works and allows us to use this same
# file on other architecutres. https://github.com/coreos/fedora-coreos-config/issues/63
#
# This file gets embedded into the efiboot.img on our Fedora CoreOS ISO.
set default="1"

function load_video {
  insmod efi_gop
  insmod efi_uga
  insmod video_bochs
  insmod video_cirrus
  insmod all_video
}

load_video
set gfxpayload=keep
insmod gzio
insmod part_gpt
insmod ext2

set timeout=5
### END /etc/grub.d/00_header ###

### BEGIN /etc/grub.d/10_linux ###
menuentry 'Fedora CoreOS (Live)' --class fedora --class gnu-linux --class gnu --class os {
	linux /images/pxeboot/vmlinuz mitigations=auto,nosmt coreos.liveiso=fedora-coreos-34.20210529.3.0 ignition.firstboot ignition.platform.id=metal
################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################ COREOS_KARG_EMBED_AREA
	initrd /images/pxeboot/initrd.img /images/ignition.img
```
