## Сборка void linux с помощью xbps-src из рабочей системы. 
Если вы любитель собирать все из исходников — эта статья прямо для Вас. Я не рекомендую использовать ее в качестве обучающего материала, если Вы не опытен. Перед прочтением моей статьи рекомендую ознакомиться с **Gentoo** **HandBook**.


#### 1. Разметка диска.
 **BIOS:**
cfdisk /dev/sdb (DOS)
 >/dev/sdb1     1G          Linux    (bootable)
 >/dev/sdb2     2×Ram  Linux swap / Solaris
 >/dev/sdb3     all          Linux

 **UEFI:**
 cfdisk /dev/sdb (GPT)
 >/dev/sdb1     1G          Efi System
 >/dev/sdb2     2×Ram  Linux filesystem
 >/dev/sdb3     all          Linux filesystem

Подробнее о том, как работает загрузка системы с MBR и с GPT вы можете узнать в этой статье:
https://habr.com/ru/post/327572/

В защиту swap раздела:
https://m.habr.com/ru/post/348324/


#### 2. Выбор файловой системы. Создаем файловую систему.
Я хочу рассмотреть 4 файловые системы: zfs, xfs, ext4, btrfs.

**ZFS** - это файловая система, объединенная с менеджером логических томов.
Файловая система ZFS имеет обычные для таких файловых систем возможности. Это просто огромный размер одного раздела, и размер файла, поддерживается возможность хранения файлов на нескольких устройствах, проверка контрольных сумм для данных и шифрование на лету, а также запись новых данных в режиме COW, когда данные не переписываются, а пишутся в новое место, что позволяет делать мгновенные снапшоты.

**XFS** - это высоко масштабируемая, высокопроизводительная файловая система, которая была изначально разработана в Silicon Graphics в 1993 году. Она была добавлена в основной состав ядра Linux в 2002 году.
Эта файловая система имеет все преимущества современных файловых систем, такие как журналирование метаданных для быстрого восстановления, но, кроме того, здесь поддерживается распределение потоков ввода/вывода по группам что сильно увеличивает производительность чтения и записи данных. Но это работает только для больших файлов. Также вы можете увеличить размер файловой системы или выполнить дефрагментацию, даже если она смонтирована.

**ext4** - очень популярная файловая система, которая используется в большинстве дистрибутивов по умолчанию. Стабильная и многофункциональная.
Плюсы:
- Журналирование
- Поддержка шифрования
- Высокая стабильность, так как она проверена временем
- Поддержка по умолчанию во многих дистрибутивах
- Активная разработка
- Не подвержена фрагментации
- Лимитов вполне достаточно обычному пользователю, так и для серверных систем

**btrfs** - это относительно новая файловая система, которая появилась в 2007 году и была разработана компанией Oracle. Она предлагает очень широкий набор новых возможностей.
Преимущества:
- Большие лимиты и хорошая масштабируемость по сравнению с Ext4
- Поддержка большинства возможностей современных файловых систем, таких как менеджер томов, сжатие на лету, дедупликация, copy-on-write, снапшоты и многое другое
- Поддержка проверки контрольных сумм, что позволяет точно обнаружить повреждение данных из-за аппаратных проблем

*btrfs* полностью пригодна для десктопа по моему мнению. И в большинстве случаев она будет наилучшим выбором. Именно поэтому я и буду использовать ее в этом "гайде".

**Создание файловой системы(DOS):**
>$ mkfs.btrfs /dev/sdb1 
>$ mkswap /dev/sdb2 -L "swap"
>$ swapon /dev/sdb2
>$ mkfs.btrfs /dev/sdb3

**Создание файловой системы(GPT):**
>$ mkfs.fat -F32 /dev/sdb1
>$ mkswap /dev/sdb2 -L "swap"
>$ swapon /dev/sdb2
>$ mkfs.btrfs /dev/sdb3

тыдэм, все готово.


#### 3. Теперь все монтируем.
># корень
>$ mkdir -p /mnt/void
>$ mount /dev/sdb3 /mnt/void

># boot раздел(DOS)
>$ mkdir -p /mnt/void/boot/
>$ mount /dev/sdb1 /mnt/void/boot
>
># boot раздел(GPT)
>$ mkdir -p /mnt/void/boot/efi
>$ mount /dev/sdb1 /mnt/void/boot/efi

>// proc, run, dev, sys
>$ mkdir -p /mnt/void/{proc,sys,dev,run}
>
>$ mount --types proc /proc /mnt/void/proc
>
>$ mount --rbind /sys /mnt/void/sys
>$ mount --make-rslave /mnt/void/sys
>
>$ mount --rbind /dev /mnt/void/dev
>$ mount --make-rslave /mnt/void/dev
>
>$ mount --bind /run /mnt/void/run
>$ mount --make-slave /mnt/void/run

**--rbind** позволяет нам получить доступ к директории из двух точек. При этом подточки будут тоже примонтированы.
**--bind** делает тоже самое, но подточки монтироваться не будут.
**--make-slave** позволяет любому монтированию, в пределах исходной точки монтирования, отражаться в ней, но ни одно монтирование, в пределах подчинённого не может отразиться в оригинале.
**--make-rslave** делает тоже самое, только он еще затрагивает подточки.

#### 4.  Компиляция необходимых пакетов.
>// установка xbps-src
>$ git clone https://github.com/void-linux/void-packages

>// подготовка
>$ xbps-install -Su base-devel
>
>$ cd void-packages
>
>$ ./xbps-src -j12 bootstrap // -j число используемых потоков
>
>// установка базовых пакетов
>$ ./xbps-src -j12 -N pkg base-minimal grub linux-firmware-intel linux-firmware-network dracut linux5.17 linux5.17-headers xbps dosfstools e2fsprogs btrfs-progs ncurses dhcpcd dbus elogind polkit rtkit pipewire alsa-pipewire opendoas
>
>$ xbps-install -R $BINPKGS -r /mnt/void base-minimal grub linux-firmware-intel linux-firmware-network dracut linux5.17 linux5.17-headers xbps dosfstools e2fsprogs btrfs-progs ncurses dhcpcd dbus elogind polkit rtkit pipewire alsa-pipewire opendoas git


#### 5. Chroot в новую систему.  Настройка конфигов.
>// копируем информацию о DNS
>$ cp --dereference /etc/resolv.conf /mnt/void/etc
>
>$ chroot /mnt/void /bin/bash
>
>$ chown root:root /
>$ chmod 755 /
>
>$ passwd root
>
>// Создаем юзера, задаем ему пароль, меняем хостнейм
>$ useradd -m -G users, wheel -s /bin/bash kiss-void
>$ passwd kiss-void
>$ echo kissofsatan > /etc/hostname
>
>// создаем и подключаем конфиг doas
>$ echo "permit keepenv persist nolog :wheel" >> /etc/doas.conf
>$ doas -C /etc/doas.conf
>
>$ echo "LANG=en_US.UTF-8" > /etc/locale.conf
>
>$ echo "en_US.UTF-8 UTF-8" >> /etc/default/libc-locales
>
>// часовой пояс
>$ ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
>
>// fstab (DOS)
>$ cat >> /etc/fstab << EOF
=> /dev/sdb1 /boot btrfs defaults, noatime 0 2
=> /dev/sdb2 none swap sw 0 0
=> /dev/sdb3 / btrfs noatime 0 1
=>EOF
>
>// fstab (GPT)
>$ cat >> /etc/fstab << EOF
>=> /dev/sdb1 /boot/efi vfat defaults, noatime 0 2
>=> /dev/sdb2 none swap sw 0 0
>=> /dev/sdb3 / btrfs noatime 0 1
>=> EOF
>
>// добавляем необходимые сервисы в автозапуск 
>$ ln -s /etc/sv/{dhcpcd,dbus,polkit} /var/service/
>
>$ xbps-reconfigure -fa


#### 6. Установка загрузчика grub.
**DOS:**

>$ grub-install /dev/sdb
>$ grub-mkconfig -o /boot/grub/grub.cfg

**GPT:**
>$ grub-install --efi-directory=/boot/efi
>$ grub-mkconfig -o /boot/grub/grub.cfg


#### 7. Завершение.
>$ exit
>
>$ umount -R /mnt/void
>
>$ reboot
