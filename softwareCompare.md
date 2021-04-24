# Сравнение программного обеспечения Fedora Core и дистрибутивов ALT

## Список ПО Fedora Core

Продукт Fedora Core | Язык | Документация |   Описание
--------------------|----------------|------|----------------------------------------------------------------
[ignition](https://github.com/coreos/ignition) | GO | [System configuration](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/) |  Утилита,  для управления дисками во время `initramfs`: настройку пользователей разбиение дисков, форматирование разделов, запись файлов (обычные файлы, файлы конфигурации сети, модули systemd и т. д.)
[ostree](https://github.com/ostreedev/ostree)| C | [docs](https://github.com/ostreedev/ostree/tree/master/docs) | Разделяемая библиотека и набор инструментов командной строки, которые объединяют «git-подобную» модель для фиксации и загрузки деревьев загрузочных файловых систем, а также уровень для их развертывания и управления конфигурацией загрузчика. Базовая модель OSTree похожа на git тем, что в ней проверяются суммы отдельных файлов и есть хранилище объектов с адресацией к содержимому. В отличие от git, он «проверяет» файлы через жесткие ссылки, и поэтому они должны быть неизменными, чтобы предотвратить повреждение. 
[butane](https://github.com/coreos/butane) | GO | [Getting started](https://github.com/coreos/butane/blob/main/docs/getting-started.md) | Переводит удобочитаемые (`YAML`) конфигурации butane  в машиночитаемые конфигурации ignition (`JSON`)
[rpm-ostree](https://github.com/coreos/rpm-ostree)| C++ | [rpm-ostree](https://coreos.github.io/rpm-ostree/)| Надстройка над ostree, позволяющая создавать пополять дерево ostree из пакетной базы дистрибутива  
[dracut](https://github.com/dracutdevs/dracut) | C | [Dracut:  Introduction and Overview](https://events.static.linuxfound.org/images/stories/pdf/lcjp2012_cong_wang.pdf), [Dracut](https://wiki.gentoo.org/wiki/Dracut)| Инструмент для создания образа initramfs путем копирования инструментов и файлов из установленной системы и объединения их с фреймворком dracut (/usr/lib/dracut/modules.d)  
[Bubblewrap](https://github.com/containers/bubblewrap/) | С | [Bubblewrap](https://wiki.archlinux.org/index.php/Bubblewrap) | setuid реализация user namespaces
[polkit](https://gitlab.freedesktop.org/polkit/polkit/) | C | [PolicyKit](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/desktop_migration_and_administration_guide/policykit) | polkit -  набор инструментов для поддержки авторизации.
[toolbox container](https://github.com/containers/toolbox) | GO | [doc](https://github.com/containers/toolbox/tree/main/doc) | Toolbox -  инструмент для операционных систем Linux, который позволяет использовать контейнерные среды командной строки. Он построен на основе Podman и других стандартных контейнерных технологий от OCI. 

### Сравнение   
Продукт Fedora Core | Наличие в sisyphus | Комментарий
---------------------|--------------------|--------------------------------------------
ignition v2.9.0 | [ignition 2.9.0-alt1](http://git.altlinux.org/gears/i/ignition.git)
ostree v2021.2 | [ostree 2020.8-alt1](http://git.altlinux.org/gears/o/ostree.git)
dracut 053 | [ dracut 053-alt1](http://git.altlinux.org/gears/d/dracut.git) | Будет ли dracut замещать make-initrd или будет использоваться совместно с ним?
Bubblewrap v0.4.1 | [Bubblewrap	0.4.1-alt2](http://git.altlinux.org/gears/b/bubblewrap.git)
polkit 0.118 | [polkit 	0.118-alt2](http://git.altlinux.org/gears/p/polkit.git?p=polkit.git;a=summary)
butane v0.11.0 | Отсутствует | Проблем с переносом быть не должно - простой конвертор форматов.
rpm-ostree v2021.4 | Отсутствует | Необходима интеграция RPM@ALTLinux и (возможно) с `mkimages-profiles`
toolbox container 0.0.99.1  | Отсутствует |
