# Адаптация

- Поддержка новой структуры каталогов файловой системы
  [Adapting existing mainstream distributions](https://ostreedev.github.io/ostree/adapting-existing/)
- Добавление ignition в make-initrd    
- Перенос (или линковка?) базы пакетов из /var/lib/rpm в /usr/lib/rpm 
- Добавление и поддержка пакета [nss-altfiles](https://github.com/aperezdc/nss-altfiles) для храниния пользоваталей и групп в дополнительных файлах usr/lib/passwd, usr/lib/group. [ref](https://coreos.github.io/rpm-ostree/administrator-handbook/#operating-system-changes)
- использование файловой истемы xfs (отсутствие /etc/fstab)

## Использование файловой истемы xfs

Судя по всему использование файловой ситемы xfs для работы с ostree эта решение Redhat.
ostree [поддерживает различные типы файловых систем](https://ostreedev.github.io/ostree/):
> Because OSTree operates at the Unix filesystem layer, it works on top of any filesystem or block storage layout; it’s possible to replicate a given filesystem tree from an OSTree repository into plain ext4, BTRFS, XFS, or in general any Unix-compatible filesystem that supports hard links. Note: OSTree will transparently take advantage of some BTRFS features if deployed on it.


## Ссылки
- **ostee**
  * [OSTree Overview](https://ostreedev.github.io/ostree/)   
- **rpm-ostree**
  * [rpm-ostree: A true hybrid image/package system](https://coreos.github.io/rpm-ostree/)
