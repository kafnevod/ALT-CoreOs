# Описание структуры файловой системы Fedora Core

## Описание дерева и ссылочкой структуры

![Описание структуры файловой системы Fedora Core](/Images/rootTree.png)

## Описание монтирования корнеых каталогов
Раздел | Каталог | Тип доступа | Тип ФС | Флаги
-------|----------|--------------|--------|-------
/dev/sda4 | /  | Чтение/Запись | xfs | rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,prjquota
/dev/sda4 |/sysroot/ | Чтение | xfs   | ro,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,prjquota
/dev/sda4 | /usr/ | Чтение | xfs   | ro,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,prjquota
/dev/sda4 | /etc/  | Чтение/Запись | xfs | rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,prjquota
/dev/sda4 | /var/  | Чтение/Запись | xfs | rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,prjquota
/dev/sda3 | /boot/ | Чтение | ext4 | ro,nosuid,nodev,relatime,seclabel
