 # Atomic Upgrades

    You can turn off the power anytime you want…
    Simple upgrades via HTTP
    Upgrades via external tools (e.g. package managers)
    Assembling a new deployment directory
    Atomically swapping boot configuration
    The bootversion
    The /ostree/boot directory
        Licensing for this document:

# Вы можете выключить питание в любое время…

OSTree предназначен для реализации полностью атомарных и безопасных обновлений; в более общем смысле, атомарные переходы между списками загрузочных развертываний. Если система выйдет из строя или вы отключите питание, у вас будет либо старая система, либо новая. 

## Простое обновление через HTTP

Прежде всего, самая базовая модель, которую поддерживает OSTree, - это модель, в которой он реплицирует предварительно сгенерированные деревья файловых систем с сервера по протоколу HTTP, отслеживая ровно одну ссылку, которая хранится в файле `.origin` для развертывания. Это реализует команда `ostree admin upgrade`.

Чтобы начать простое обновление, OSTree получает содержимое ссылки с удаленного сервера. Предположим, мы отслеживаем ссылку с именем `exampleos/buildmaster/x86_64-runtime`. OSTree получает URL `http://example.com/repo/refs/heads/exampleos/buildmaster/x86_64-runtime`, который содержит контрольную сумму SHA256. Это определяет дерево для развертывания, и `/etc` будет объединен с загруженным в данный момент деревом.

Если у нас нет этого коммита, мы выполняем процесс извлечения. В настоящее время (без статических дельт) для этого достаточно просто получить каждый отдельный объект, которого у нас нет, в асинхронном режиме. Другими словами, мы загружаем только измененные файлы (сжатые zlib). Контрольная сумма каждого объекта проверена и хранится в `/ostree/repo/objects/`. 

После завершения извлечения мы загрузи все объекты, необходимые для развертывания. 

## Обновления с помощью внешних инструментов (например, менеджеров пакетов)

Как упоминалось во введении, OSTree также поддерживает модель, в которой деревья файловых систем формируются на клиенте. 
Совершенно безразлично, как эти деревья генерируются; они могут быть сформированы с помощью традиционных пакетов, пакетов с сценариями после развертывания из корневого директория или построены разработчиками непосредственно из локального контроля версий и т. д.

На практическом уровне большинство менеджеров пакетов сегодня (`dpkg` и `rpm`) работают «вживую» с файловой системой, загруженной в данный момент. Вместо этого они могли бы работать с OSTree, взяв список установленных пакетов в текущем загруженном дереве и сформировав из него новую файловую систему. В следующей главе более подробно описывается, как это может работать: [Адаптация существующих систем](distributions.md).

Для целей этого раздела предположим, что у нас есть вновь сгенерированное дерево файловой системы, хранящееся в репозитории (которое разделяет хранилище с существующим загруженным деревом). 
Затем мы можем перейти к проверке его из репозитория в развертывание. 

## Сборка нового каталога развертывания

Получив коммит для развертывания, OSTree сначала выделяет для него каталог. 
Он имеет вид `/boot/loader/entries/ostree-$stateroot-$checksum.$serial.conf`. `$serial` обычно равен 0, но если данная фиксация развернута более одного раза, она будет увеличиваться. 
Это делается для того, чтобы предыдущее развертывание могло иметь конфигурацию в `/etc`, которую мы не хотим использовать или перезаписывать.

Теперь, когда у нас есть каталог развертывания, выполняется трехстороннее слияние между (по умолчанию) загруженным в данный момент развертыванием `/etc`, его конфигурацией по умолчанию и новым развертыванием (на основе его `/usr/etc`).

Как это работает:
- Файлы в текущем загруженном развертывании `/etc`, которые были изменены из `/usr/etc` по умолчанию (того же развертывания), сохраняются.
- Файлы в текущем загруженном развертывании `/etc`, которые не были изменены из `/usr/etc` по умолчанию (того же развертывания), обновляются до новых значений по умолчанию из `/usr/etc`? нового развертывания.

Проще говоря, это означает, что как только вы изменяете или добавляете файл в `/etc`, этот файл будет зафиксирован на все последующие развертывания 
(хотя есть крайний случай, когда ваша модификация в конечном итоге точно соответствует будущему файлу по умолчанию, тогда файл с этого момента будет обновляться в последующих развертываниях).

Вы можете использовать команду `ostree admin config-diff`, чтобы увидеть различия между загруженным развертыванием `/etc` и настройками по умолчанию в OSTree. 


## Atomically swapping boot configuration

At this point, a new deployment directory has been created as a hardlink farm; the running system is untouched, and the bootloader configuration is untouched. We want to add this deployment to the “deployment list”.

To support a more general case, OSTree supports atomic transitioning between arbitrary sets of deployments, with the restriction that the currently booted deployment must always be in the new set. In the normal case, we have exactly one deployment, which is the booted one, and we want to add the new deployment to the list. A more complex command might allow creating 100 deployments as part of one atomic transaction, so that one can set up an automated system to bisect across them.

## The bootversion

OSTree allows swapping between boot configurations by implementing the “swapped directory pattern” in /boot. This means it is a symbolic link to one of two directories /ostree/boot.[0|1]. To swap the contents atomically, if the current version is 0, we create /ostree/boot.1, populate it with the new contents, then atomically swap the symbolic link. Finally, the old contents can be garbage collected at any point.

## The /ostree/boot directory

However, we want to optimize for the case where the set of kernel/initramfs/devicetree sets is the same between both the old and new deployment lists. This happens when doing an upgrade that does not include the kernel; think of a simple translation update. OSTree optimizes for this case because on some systems /boot may be on a separate medium such as flash storage not optimized for significant amounts of write traffic. Related to this, modern OSTree has support for having /boot be a read-only mount by default - it will automatically remount read-write just for the portion of time necessary to update the bootloader configuration.

To implement this, OSTree also maintains the directory /ostree/boot.$bootversion, which is a set of symbolic links to the deployment directories. The $bootversion here must match the version of /boot. However, in order to allow atomic transitions of this directory, this is also a swapped directory, so just like /boot, it has a version of 0 or 1 appended.

Each bootloader entry has a special ostree= argument which refers to one of these symbolic links. This is parsed at runtime in the initramfs.
