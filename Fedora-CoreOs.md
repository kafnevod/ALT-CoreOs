# Описание дистрибутива Fedore CoreOs

Отличительными особенностями дистрибутива Fedore CoreOs являются:
- отсутствие этапа "ручной" настройки. Все нижеописанные настройки задаются в файле в формате ignition передаваемому параметром команде формирования загрузочного образа дистрибутива `coreos-installer` 
  


## Ignition

Файл формата ignition (суффикс .ign) обеспечивает:
- разбиение дисков на партиции (storag.disks.device.patrition, storage.filesystems);
 - создание пользователей (passwd.users) и групп (passwd.groups);
  - запуск и конфигурирование сервисов (systems.units) включая docker-контейнеры;
  - добавление или замена директориев (storage.directories) и(конфигурационных) файлов (storage.files, storage.links), что позволяет сконфигурировать:
    * сетевые интерфейсы;  
    * параметры ядра (rpm-ostree kargs ..., storage.files.path=/etc/sysctl.d/..., /etc/sysctl.d/...).
