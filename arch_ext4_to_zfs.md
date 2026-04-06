# Инструкция по перемещению рабочей операционной системы Arch Linux с Ext4 на ZFS

### Дисклеймер

В принципе - данная инструкция подойдёт для любого дистрибутива, если вы знаете тонкости управления загрузчиком и процессом загрузки

[Поиграйте](./play_with_zfs.md) заранее на своей системе с файлами в качестве vdev.

## Шаг 1: Подготовка

 1. Внимательно изучите расположение примонтированных разделов с помощью команд `df -h`, `lsblk`, `parted`, `ls -l /dev/disk/by-id/`. Запишите на бумаге как можно больше информации, составьте дерево вложенности примонтированных устройств и их id.

 1. Максимально облегчите систему, узнать, что занимает больше всего места поможет команда `du -sh /path/*`

    * Перенесите большие файлы, такие как видео, музыку и фотографии на отдельное хранилище на время миграции для сокращения времени работ. 

    * Очистите кэши

 1. Создайте полный бекап системы с помощью LiveCD Clonezilla или утилиты `dd` на отдельном носителе или в файловом хранилище

 1. Загрузитесь в LiveCD, я использовал [Debian trixie live xfce4](https://mirror.yandex.ru/debian-cd/13.4.0-live/amd64/iso-hybrid/debian-live-13.4.0-amd64-xfce.iso)

    * Для удобства дальшейшей работы (если у вас есть второй ПК или ноутбук) установите SSH

      ```shell
      sudo apt update
      sudo apt install --yes openssh-server
      sudo systemctl restart ssh
      ```

    * Узнайте IP Live-системы

      ```shell
      ip a | grep inet
      ```

    * В Debian Live пользователь `user` с паролем `live`

 1. Создайте полный архив файловой системы рабочей ОС 

    * Примонтируйте раздел, содержащий "/" вашей системы в "/mnt/" Live-системы

    * Примонтируйте остальные разделы вашей системы в нужные директории с префиксом "/mnt/"

    * Создайте директорию /backup и примонтируйте туда отдельный физический носитель достаточного объёма

    * Склонируйте файловую систему с помощью команды

      ```shell
      rsync -aAXH /mnt/ /backup/
      sync
      ```
      Обратите внимание на закрывающие слэши в путях команды rsync

 1. Отмонтируйте ФС

    ```shell
    umount /backup
    umount -R /mnt
    ```

 1. Установите поддержку ZFS в Live-систему (по мотивам [этой](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/index.html#installation) и [этой](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Trixie%20Root%20on%20ZFS.html#step-1-prepare-the-install-environment) инструкций) . Все операции производятся от имени суперпользователя (root).

    * Добавьте репозиторий backports и установите приоритет для поддержки ZFS

      ```shell
      cat > /etc/apt/sources.list.d/trixie-backports.list << EOF
      deb http://deb.debian.org/debian trixie-backports main contrib non-free-firmware
      deb-src http://deb.debian.org/debian trixie-backports main contrib non-free-firmware
      EOF

      cat > /etc/apt/preferences.d/90_zfs << EOF
      Package: src:zfs-linux
      Pin: release n=trixie-backports
      Pin-Priority: 990
      EOF
      ```
    * Установите необходимые пакеты

      ```shell
      apt update
      apt install --yes linux-headers-generic
      apt install --yes gdisk zfsutils-linux
      ```

      Обратите внимание, так как лицензия CDDL (Common Development and Distribution License), под которой распространяется OpenZFS несовместима с GNU GPL (General Public License), необходимо принять соглашение на использование данного ПО. Если вы несогласны - завершите работу и продолжайте наслаждаться своей ОС.

## Шаг 2:  Форматирование диска

*В данной инструкции не рассматривается шифрование LUKS (только ZFS-native) и создание RAID-массивов в течении миграции ОС, освещение добавления дисков в зеркальный RAID-массив приводится в завершающем этапе статьи, а примеры частных случаев шифрования LUKS и RAIDz рекомендую использовать только профессионалам и обратиться к приведённым в разделах ссылкам.*

*Для получения информации о приведённых в статье параметрах ФС ZFS также предлагаю использовать статьи по ссылкам в разделах, либо справочной документации ZFS*

 1. Установите переменную с длинным id диска и переменную с именем пользователя (не вздумайте пытаться менять имя пользователя на лету!). Всегда используйте имена дисков в ФС ZFS для предотвращения сбоев.

    * Получаем список дисков и копируем наш. Нас интересуют только диски без окончания "-partX"

      ```shell
      ls -la /dev/disk/by-id/
      ```

    * Устанавливаем пременную $DISK

      ```shell
      DISK=/dev/disk/by-id/ID-ВАШЕГО-ДИСКА
      ```

    * Установим и переменную $MYUSER. Вместо username напишите имя вашего пользователя из переносимой системы. Если пользователей больше одного, то мы это поправим на этапе создания датасетов.

      ```shell
      MYUSER=username
      ```

 1. Так как диск не новый - его нужно очистить. ВНИМАНИЕ! ДИСК БУДЕТ ОЧИЩЕН БЕЗВОЗВРАТНО!

    * Выключаем swap

      ```shell
      swapoff --all
      ```

    * Если использовался массив MD, он удаляется специальными командами:

      ```shell
      apt install --yes mdadm
      # Смотрим, есть ли хоть один активный массив MD
      cat /proc/mdstat
      # Если есть - остановим их (замените md0 на нужный)
      mdadm --stop /dev/md0
      
      # Для массива, использующего весь диск:
      mdadm --zero-superblock --force $DISK
      # Для массива, использующего раздел:
      mdadm --zero-superblock --force ${DISK}-part2
      ```

    * Далее очистим диск от других вариантов, некоторые из команд могут вызвать ошибки, это всего лишь значит, что данный метод очистки не нашёл следов нужного ему раздела.

      ```shell
      wipefs -a $DISK
      blkdiscard -f $DISK
      sgdisk --zap-all $DISK
      partprobe $DISK
      ```

    * Создадим разделы 

      ```shell
      sgdisk -a1 -n1:24K:+1000K -t1:EF02 $DISK
      sgdisk     -n2:1M:+512M   -t2:EF00 $DISK
      sgdisk     -n3:0:+1G      -t3:BF01 $DISK
      sgdisk     -n4:0:0        -t4:BF00 $DISK
      ```

    * Создадим загрузочный пул

      ```shell
      zpool create \
          -o ashift=12 \
          -o autotrim=on \
          -o compatibility=grub2 \
          -o cachefile=/etc/zfs/zpool.cache \
          -O devices=off \
          -O acltype=posixacl -O xattr=sa \
          -O compression=lz4 \
          -O normalization=formD \
          -O relatime=on \
          -O canmount=off -O mountpoint=/boot -R /mnt \
          zboot ${DISK}-part3
      ```

    * Создадим корневой пул

      * БЕЗ шифрования:

        ```shell
        zpool create \
            -o ashift=12 \
            -o autotrim=on \
            -O acltype=posixacl -O xattr=sa -O dnodesize=auto \
            -O compression=lz4 \
            -O normalization=formD \
            -O relatime=on \
            -O canmount=off -O mountpoint=/ -R /mnt \
            zroot ${DISK}-part4
        ```

      * С шифрованием ZFS-native

        ```shell
        zpool create \
            -o ashift=12 \
            -o autotrim=on \
            -O encryption=on -O keylocation=prompt -O keyformat=passphrase \
            -O acltype=posixacl -O xattr=sa -O dnodesize=auto \
            -O compression=lz4 \
            -O normalization=formD \
            -O relatime=on \
            -O canmount=off -O mountpoint=/ -R /mnt \
            zroot ${DISK}-part4
        ```

## Шаг 3: Создание датасетов

 1. Создание основных датасетов-контейнеров

    ```shell
    zfs create -o canmount=off -o mountpoint=none zroot/ROOT
    zfs create -o canmount=off -o mountpoint=none zboot/BOOT
    ```

 1. Создание и монтирование датасетов для монтирования корневой директории и директории /boot

    ```shell
    zfs create -o canmount=noauto -o mountpoint=/ zroot/ROOT/arch
    zfs mount zroot/ROOT/arch
    
    zfs create -o mountpoint=/boot zboot/BOOT/arch
    ```

 1. Создание датасетов для основных системных директорий

    ```shell
    zfs create                           zroot/home
    zfs create -o mountpoint=/root       zroot/home/root
    chmod 700 /mnt/root
    zfs create -o mountpoint=/home/$MYUSER zroot/home/$MYUSER
    zfs create -o canmount=off           zroot/var
    zfs create -o canmount=off           zroot/var/lib
    ```

    Если у вас больше одного пользователя - создайте каждому из них по датасету, явно указывая имена пользователей. Права раздадим уже из-под chroot.

 1. Создание второстепенных датасетов.

    *Используется в основном для облегчения управления снапшотами и автоматизации.*

    * Логирование и спул печати:

      ```shell
      zfs create                     zroot/var/log
      zfs create                     zroot/var/spool
      ```

    * Например, можно создать датасет с опцией, отключающей создание авто-снапшотов:

      ```shell
      zfs create -o com.sun:auto-snapshot=false zroot/var/cache
      zfs create -o com.sun:auto-snapshot=false zroot/var/lib/nfs
      ```

    * При использовании Docker рекомендуется выделить для него отдельный датасет

      ```shell
      zfs create -o com.sun:auto-snapshot=false zroot/var/lib/docker
      ```

Вы можете создать сколько угодно датасетов по своему усмотрению, посмотреть подсказки можно в [этом источнике](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Trixie%20Root%20on%20ZFS.html#step-3-system-installation), однако, не стоит увлекаться. Рекомендую продумать этот момент и вынести на бумагу. Датасет не равно директории, помните об этом. Также не забывайте, что для каждого шага вложенности должен быть создан датасет предыдущего этапа.

## Шаг 4: Восстановление системы на ФС ZFS

 1. Синхронизируем директории "/backup" и "/mnt", предварительно примонтировав первую

    ```shell
    mount /ВАШ_ДИСК_С_БЕКАПОМ_СИСТЕМЫ /backup
    df -h # Проверим, правильно ли примонтированы датасеты ZFS
    rsync -aAXH /backup/ /mnt/
    sync
    ```

 1. Примонтируем "/run" с ФС tmpfs

    ```shell
    mkdir -p /mnt/run
    mount -t tmpfs tmpfs /mnt/run
    mkdir /mnt/run/lock
    ```

 1. Примонтируем виртуальные ФС и перейдём в chroot вашей системы

    ```shell
    mount --make-private --rbind /dev  /mnt/dev
    mount --make-private --rbind /proc /mnt/proc
    mount --make-private --rbind /sys  /mnt/sys
    chroot /mnt /usr/bin/env DISK=$DISK bash --login
    ``` 

 1. Изменим хозяина папки пользователя (для каждого из пользователей) вместо "user" подставьте имя существующего пользователя

    ```shell
    chown user:user /home/user
    ```

 1. Установим ZFS, если этого не было сделано до начала процесса миграции. Я пользуюсь помощником пакетного менеджера yay, если вы пользуетесь другим помощниклм - ипользуйте его, либо воспользуйтесь AUR. Также, как видно из следующей комманды, я использую ядро LTS, если вы используете другое - необходимо устанавливать [ZFS для вашего ядра](https://wiki.archlinux.org/title/ZFS#Installation). 

    ```shell
    yay -S zfs-linux-lts zfs-utils
    ```

    Если хотите ускорить установку и упростить последующие обновления - вы можете воспользоваться специальным репозиторием archzfs и производить установку оттуда.

    ```shell
    cat >> /etc/pacman.conf << EOF
    [archzfs]
    SigLevel = Required
    Server = https://github.com/archzfs/archzfs/releases/download/experimental
    EOF

    pacman-key --init
    pacman-key --recv-keys 3A9917BF0DED5C13F69AC68FABEC0A1208037BE9
    pacman-key --lsign-key 3A9917BF0DED5C13F69AC68FABEC0A1208037BE9

    pacman -Syy
    ```

    Тогда, комманда установки будет выглядеть следующим образом:

    ```shell
    pacman -S archzfs-linux-lts
    ```

 1. Теперь пропишем hostid в кэш zfs и установим этот кэш для пулов (это требуется для правильной работы ФС)

    ```shell
    zgenhostid
    zpool set cachefile=/etc/zfs/zpool.cache zboot
    zpool set cachefile=/etc/zfs/zpool.cache zroot
    ```

 1. Для загрузки модуля ZFS при загрузке системы нам необходимо проделать некоторые изменения:

    * Отредактируйте строку "HOOKS" в файле `/etc/mkinitcpio.conf`, добавив запись "zfs" _перед_ "filesystems"

      ```shell
      HOOKS=(... ... zfs filesystems ...)
      ```

    * Также добавление модуля в систему под chroot требует явного указания версии ядра, сначала узнаем её

      ```shell
      ls /lib/modules
      ```

    * Запустим размещение модуля

      ```shell
      mkinitcpio -k 6.18.21-1-lts -g /boot/initramfs-linux-lts.img
      ```

 1. Настроим загрузчик GRUB

    * Отредактируем `/etc/default/grub`, нужно убрать запись "rootfstype=ext4" и добавить `zfs=zroot/ROOT/arch rw` в строке GRUB_CMDLINE_LINUX

    * Обновим конфиг

      ```shell
      grub-mkconfig -o /boot/grub/grub.cfg
      ```

 1. Установим GRUB в загрузочную область диска

    * Для систем без UEFI (загрузка BIOS)

      ```shell
      grub-install $DISK
      ```

    * Для систем с UEFI

      ```shell
      grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian --recheck
      ```

 1. Закомментируйте строки с "/", "/home", "/boot" и прочими директориями, кроме внешних или явно необходимых в файле `/etc/fstab`. ZFS примонтирует их сам при загрузке.

 1. Пропишите нужные сервисы в загрузку

    ```shell
    systemctl enable zfs-import-cache zfs-mount zfs-import.target zfs.target
    ```

 1. Изменим параметры монтирования пулов

    Нам с вами нужно активировать `zfs-mount-generator`. Это предотвратит раздельное монтирование датасетов из systemdбчто важно, например, для `/var/log` и `/var/tmp`. В свою очередь - `rsyslog.service` зависит от `var-log.mount` через `local-fs.target` и сервисы, автоматически использующие `After=var-tmp.mount` возможность `PrivateTmp` из systemd.

    ```shell
    mkdir /etc/zfs/zfs-list.cache
    touch /etc/zfs/zfs-list.cache/zboot
    touch /etc/zfs/zfs-list.cache/zroot
    zed -F &
    ```

    Проверьте, что `zed` обновила кэш, заполнив файлы

    ```shell
    cat /etc/zfs/zfs-list.cache/zboot
    cat /etc/zfs/zfs-list.cache/zroot
    ```

    В том случае, если файлы пусты - заставьте кэш переформироваться заново 

    ```shell
    zfs set canmount=on     zboot/BOOT/arch
    zfs set canmount=noauto zroot/ROOT/arch
    ```

    Если файлы всё ещё пусты остановите (как показано ниже) и перезапустите (как показано выше) `zed` и проверьте ещё.

    Когда данные появятся остановите `zed`

    ```shell
    fg
    ```

    нажмите CTRL+C

    Исправим пути, удалив `/mnt`

    ```shell
    sed -Ei "s|/mnt/?|/|" /etc/zfs/zfs-list.cache/*
    ```

# Шаг 5: Выход и перезагрузка

Критически важно правильно выйти и отмонтировать пулы

```shell
exit
umount -R /mnt
zpool export zboot
zpool export zroot
reboot
```

Если на каком-то из этапов экспорта пулов возникнет ошибка о том, что пул занят - придётся при загрузке импортировать их вручную из emergency shell  с помощью комманды

```shell
zpool import -f zroot # для пула zroot
```

# Если что-то пошло не так

Или как вернуться в chroot системы, которая не загружается

 1. Загрузитесь обратно в LiveCD и установите zfs (как в Шаг 1)

 1. Примонтируйте корректно все пулы

    ```shell
    zpool export -a
    zpool import -N -R /mnt zroot
    zpool import -N -R /mnt zboot
    zfs load-key -a
    zfs mount zroot/ROOT/arch
    zfs mount -a
    ```

 1. Войдите в систему с помощью chroot

    ```shell
    mount --make-private --rbind /dev  /mnt/dev
    mount --make-private --rbind /proc /mnt/proc
    mount --make-private --rbind /sys  /mnt/sys
    mount -t tmpfs tmpfs /mnt/run
    mkdir /mnt/run/lock
    chroot /mnt /bin/bash --login
    mount /boot/efi
    mount -a
    ```

 1. Проделайте необходимы вам операции

 1. После окончания работ - корректно всё отключите

    ```shell
    exit
    mount | grep zfs | tac | awk '/\/mnt/ {print $3}' | xargs -i{} umount -lf {}
    zpool export -a
    reboot
    ```

### На этом всё! 

Желаю вам удачной миграции. При больших проблемах вы всегда можете вернуться на Ext4, создав разделы заново. Вы же оставили записи на бумаге? 
