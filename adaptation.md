# Адаптация

- Поддержка новой структуры каталогов файловой системы
  [Adapting existing mainstream distributions](https://ostreedev.github.io/ostree/adapting-existing/)
- Добавление ignition в make-initrd    
- Перенос (или линковка?) базы пакетов из /var/lib/rpm в /usr/lib/rpm 
- Добавление и поддержка пакета [nss-altfiles](https://github.com/aperezdc/nss-altfiles) для храниния пользоваталей и групп в дополнительных файлах usr/lib/passwd, usr/lib/group. [ref](https://coreos.github.io/rpm-ostree/administrator-handbook/#operating-system-changes)
- использование файловой истемы xfs (отсутствие /etc/fstab)


## Ссылки
- **ostee**
  * [OSTree Overview](https://ostreedev.github.io/ostree/)   
- **rpm-ostree**
  * [rpm-ostree: A true hybrid image/package system](https://coreos.github.io/rpm-ostree/)
