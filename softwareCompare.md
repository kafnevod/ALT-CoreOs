# Сравнение программного обеспечения Fedora Core и дистрибутивов ALT

## Список ПО Fedora Core

Продукт Fedora Core | Язык | Документация |   Описание
--------------------|----------------|------|----------------------------------------------------------------
[ignition](https://github.com/coreos/ignition) | GO | [System configuration](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/) |  Утилита,  для управления дисками во время `initramfs`: настройку пользователей разбиение дисков, форматирование разделов, запись файлов (обычные файлы, файлы конфигурации сети, модули systemd и т. д.)
[ostree](https://github.com/ostreedev/ostree)| C | [docs](https://github.com/ostreedev/ostree/tree/master/docs) | Разделяемая библиотека и набор инструментов командной строки, которые объединяют «git-подобную» модель для фиксации и загрузки деревьев загрузочных файловых систем, а также уровень для их развертывания и управления конфигурацией загрузчика. Базовая модель OSTree похожа на git тем, что в ней проверяются суммы отдельных файлов и есть хранилище объектов с адресацией к содержимому. В отличие от git, он «проверяет» файлы через жесткие ссылки, и поэтому они должны быть неизменными, чтобы предотвратить повреждение. 
[dracut](https://github.com/dracutdevs/dracut) | C | [Dracut:  Introduction and Overview](https://events.static.linuxfound.org/images/stories/pdf/lcjp2012_cong_wang.pdf), [Dracut](https://wiki.gentoo.org/wiki/Dracut)| Инструмент для создания образа initramfs путем копирования инструментов и файлов из установленной системы и объединения их с фреймворком dracut (/usr/lib/dracut/modules.d) 
[butane](https://github.com/coreos/butane) | GO | [Getting started](https://github.com/coreos/butane/blob/main/docs/getting-started.md) | Переводит удобочитаемые (`YAML`) конфигурации butane  в машиночитаемые конфигурации ignition (`JSON`)
[rpm-ostree](https://github.com/coreos/rpm-ostree)| C++ | [rpm-ostree](https://coreos.github.io/rpm-ostree/)| Надстройка над ostree, позволяющая создавать пополять дерево ostree из пакетной базы дистрибутива  
  
### Сравнение   
Продукт Fedora Core | Наличие в sisyphus | Комментарий
---------------------|--------------------|--------------------------------------------
ignition v2.9.0 | [ignition 2.9.0-alt1](http://git.altlinux.org/gears/i/ignition.git)
ostree v2021.2 | [ostree 2020.8-alt1](http://git.altlinux.org/gears/o/ostree.git)
butane v0.11.0 | Отсутствует | Проблем с переносом быть не должно - простой конфртор форматов.
rpm-ostree v2021.4 | Отсутствует | Необходима интеграция RPM@ALTLinux и (возможно) с `mkimages-profiles`
