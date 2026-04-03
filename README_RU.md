# Инструкция по перемещению рабочей операционной системы Arch Linux с Ext4 на ZFS

### Дисклеймер

В принципе - данная инструкция подойдёт для любого дистрибутива, если вы знаете тонкости управления загрузчиком и процессом загрузки

## Шаг 1: Подготовка

 1. Внимательно изучите расположение примонтированных разделов с помощью команд `df -h`, `lsblk`, `parted`, `ls -l /dev/disk/by-id/`. Запишите на бумаге как можно больше информации, составьте дерево вложенности примонтированных устройств и их id.

 1. Максимально облегчите систему, узнать, что занимает больше всего места поможет команда `du -sh /path/*`

    * Перенесите большие файлы, такие как видео, музыку и фотографии на отдельное хранилище на время миграции для сокращения времени работ. 

    * Очистите кэши

 1. Создайте полный бекап системы с помощью LiveCD Clonezilla или утилиты `dd` на отдельном носителе или в файловом хранилище

 1. Загрузитесь в графический LiveCD, я использовал [Debian trixie live xfce4](https://mirror.yandex.ru/debian-cd/13.4.0-live/amd64/iso-hybrid/debian-live-13.4.0-amd64-xfce.iso)

 1. Создайте полный архив файловой системы рабочей ОС 

    * Примонтируйте раздел, содержащий "/" вашей системы в "/mnt/" Live-системы

    * Примонтируйте остальные разделы вашей системы в нужные директории с префиксом "/mnt/"

    * Создайте диреторию /backup и примонтируйте туда отдельный физический носитель достаточного объёма

    * Склонируйте файловую систему с помощью команды

      ```shell
      rsync -aAXH /mnt/ /backup/ --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"}
      sync
      ```
      Обратите внимание на закрывающие слэши в путях команды

 1. Установите поддержку ZFS в Live-систему (по мотивам (этой)[https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/index.html#installation] и (этой)[https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Trixie%20Root%20on%20ZFS.html#step-1-prepare-the-install-environment] инструкций) . Все операции производятся от имени суперпользователя (root).

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

## Шаг 2:  Форматирование диска

*В данной инструкции не рассматривается шифрование LUKS (только ZFS-native) и создание RAID-массивов в течении миграции ОС, освещение добавления дисков в зеркальный RAID-массив приводится в завершающем этапе статьи, а примеры частных случаев шифрования LUKS и RAIDz рекомендую использовать только профессионалам и обратиться к приведённым в разделах ссылкам.*

*Для получения информации о приведённых в статье параметрах ФС ZFS также предлагаю использовать статьи по ссылкам в разделах, либо справочной документации ZFS*

 1. Установите переменную с длинным id диска. Всегда используйте имена дисков в ФС ZFS для предотвращения сбоев.

    * Получаем список дисков и копируем наш. Нас интересуют только диски без окончания "-partX"

      ```shell
      ls -la /dev/disk/by-id/
      ```

    * Устанавливаем пременную $DISK

      ```shell
      DISK=/dev/disk/by-id/ID-ВАШЕГО-ДИСКА
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
