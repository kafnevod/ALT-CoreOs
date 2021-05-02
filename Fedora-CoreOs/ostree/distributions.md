# Адаптация существующих основных дистрибутивов 
    System layout
    Booting and initramfs technology
    /usr/lib/passwd
    Adapting existing package managers
        Licensing for this document:

## Схема файловой системы

Прежде всего, OSTree стимулирует системы внедрять UsrMove. 
Данный подход предлагается для того, чтобы избежать необходимости в дополнительных `bind mounts`. 
По умолчанию `dracut hook` в OSTree создает `bind mounts` в режиме только для чтения только для каталога `/usr`; 
вы, конечно, можете создать отдельные bind-mounts для `/bin`, всех вариантов `/lib` и т. д. 
Так что это не является жестким требованием.

Помните, поскольку по умолчанию система загружается в chroot-эквиваленте, должен быть какой-то способ обратиться к реальной физической корневой файловой системе. 
Следовательно, дерево вашей операционной системы должно содержать пустой каталог `/sysroot`; 
во время загрузки OSTree сделает через bind-mount этот каталог привязанным к физическому / корневому каталогу. 
Есть прецедент использования этого имени в контексте initramfs. 

Кроме того, вы должны создать символическую ссылку `/ostree` верхнего уровня, которая указывает на `/sysroot/ostree`, чтобы инструмент OSTree во время выполнения мог постоянно находить системные данные независимо от того, работает ли он в физическом корне или внутри развертывания.

Поскольку OSTree при обновлении сохраняет только `/var`  (chroot-каталог каждого развертывания в конечном итоге будет собираться мусором), вам нужно будет выбрать, как поддерживать другие доступные для записи каталоги верхнего уровня, указанные в [Стандарте иерархии файловой системы](https://www.pathname.com/fhs/). 
Ваша операционная система, конечно, может не поддерживать некоторые из них, такие как `/usr/local`, но рекомендуется следующий набор: 

    /home → /var/home
    /opt → /var/opt
    /srv → /var/srv
    /root → /var/roothome
    /usr/local → /var/usrlocal
    /mnt → /var/mnt
    /tmp → /sysroot/tmp

Более того, поскольку `/var` по умолчанию пуст, ваша операционная система должна будет динамически создавать их целевые объекты при загрузке. 
Хороший способ сделать это - использовать systemd-tmpfiles, если ваша ОС использует systemd. Например: 
- d /var/log/journal 0755 root root -
- L /var/home - - - - ../sysroot/home
- d /var/opt 0755 root root -
- d /var/srv 0755 root root -
- d /var/roothome 0700 root root -
- d /var/usrlocal 0755 root root -
- d /var/usrlocal/bin 0755 root root -
- d /var/usrlocal/etc 0755 root root -
- d /var/usrlocal/games 0755 root root -
- d /var/usrlocal/include 0755 root root -
- d /var/usrlocal/lib 0755 root root -
- d /var/usrlocal/man 0755 root root -
- d /var/usrlocal/sbin 0755 root root -
- d /var/usrlocal/share 0755 root root -
- d /var/usrlocal/src 0755 root root -
- d /var/mnt 0755 root root -
- d /run/media 0755 root root -

Особо обратите внимание на двойную переадресацию на `/home`. 
По умолчанию каждое развертывание будет совместно использовать глобальный каталог верхнего уровня `/home` в физической корневой файловой системе. 
Затем инструменты управления более высокого уровня должны поддерживать синхронизацию `/etc/passwd` или его эквивалента между операционными системами. 
Каждое развертывание можно легко перенастроить, чтобы установить собственный домашний каталог, просто сделав `/var/home` настоящим каталогом.

## Booting and initramfs technology

OSTree comes with optional dracut+systemd integration code which follows this logic:

    Parse the ostree= kernel command line argument in the initramfs
    Set up a read-only bind mount on /usr
    Bind mount the deployment’s /sysroot to the physical /
    Use mount(MS_MOVE) to make the deployment root appear to be the root filesystem

After these steps, systemd switches root.

If you are not using dracut or systemd, using OSTree should still be possible, but you will have to write the integration code. See the existing sources in src/switchroot as a reference.

Patches to support other initramfs technologies and init systems, if sufficiently clean, will likely be accepted upstream.

A further specific note regarding sysvinit: OSTree used to support recording device files such as the /dev/initctl FIFO, but no longer does. It’s recommended to just patch your initramfs to create this at boot.

## /usr/lib/passwd

Unlike traditional package systems, OSTree trees contain numeric uid and gids. Furthermore, it does not have a %post type mechanism where useradd could be invoked. In order to ship an OS that contains both system users and users dynamically created on client machines, you will need to choose a solution for /etc/passwd. The core problem is that if you add a user to the system for a daemon, the OSTree upgrade process for /etc will simply notice that because /etc/passwd differs from the previous default, it will keep the modified config file, and your new OS user will not be visible. The solution chosen for the Gnome Continuous operating system is to create /usr/lib/passwd, and to include a NSS module nss-altfiles which instructs glibc to read from it. Then, the build system places all system users there, freeing up /etc/passwd to be purely a database of local users. See also a more recent effort from Systemd Stateless

## Adapting existing package managers

The largest endeavor is likely to be redesigning your distribution’s package manager to be on top of OSTree, particularly if you want to keep compatibility with the “old way” of installing into the physical /. This section will use examples from both dpkg and rpm as the author has familiarity with both; but the abstract concepts should apply to most traditional package managers.

There are many levels of possible integration; initially, we will describe the most naive implementation which is the simplest but also the least efficient. We will assume here that the admin is booted into an OSTree-enabled system, and wants to add a set of packages.

Many package managers store their state in /var; but since in the OSTree model that directory is shared between independent versions, the package database must first be found in the per-deployment /usr directory. It becomes read-only; remember, all upgrades involve constructing a new filesystem tree, so your package manager will also need to create a copy of its database. Most likely, if you want to continue supporting non-OSTree deployments, simply have your package manager fall back to the legacy /var location if the one in /usr is not found.

To install a set of new packages (without removing any existing ones), enumerate the set of packages in the currently booted deployment, and perform dependency resolution to compute the complete set of new packages. Download and unpack these new packages to a temporary directory.

Now, because we are merely installing new packages and not removing anything, we can make the major optimization of reusing our existing filesystem tree, and merely layering the composed filesystem tree of these new packages on top. A command like this:
```
ostree commit -b osname/releasename/description \
    --tree=ref=$osname/$releasename/$description \
    --tree=dir=/var/tmp/newpackages.13A8D0/
```
will create a new commit in the $osname/$releasename/$description branch. The OSTree SHA256 checksum of all the files in /var/tmp/newpackages.13A8D0/ will be computed, but we will not re-checksum the present existing tree. In this layering model, earlier directories will take precedence, but files in later layers will silently override earlier layers.

Then to actually deploy this tree for the next boot: ostree admin deploy $osname/$releasename/$description

This is essentially what rpm-ostree does to support its package layering model.
