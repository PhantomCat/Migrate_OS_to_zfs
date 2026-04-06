# Поиграйте с ZFS!

Поиграйте с ZFS заранее на своей системе с файлами в качестве vdev:

1. Установите на своей системе [ZFS](https://wiki.archlinux.org/title/ZFS#Installation)

1. Поиграйте с ZFS, чтобы понять основные принципы.

   * перейдём в домашнюю директорию (для суперпользователя это /root)

   ```shell
   cd
   ```
   
   * создадим пустые файлы

   ```shell
   touch file{1,2}
   ```
   
   * забронируем место (по 10 МегаБайт)

   ```shell
   fallocate -l 1G file1
   fallocate -l 1G file2
   ```
   
   * создадим пул с точкой монтирования в домашней директории

   ```shell
   sudo zpool create -O mountpoint=${HOME}/tank tank ./file1
   ```
   
   * ИНТЕРЕСНЫЙ МОМЕНТ - добавим второй файл к пулу в качестве второго участника зеркального массива (RAID1)

   ```shell
   sudo zpool attach tank ${HOME}/file1 ./file2
   ```
   
   * создадим датасеты внутри пула, они автоматически примнтируются как подпапки

   ```shell
   sudo zfs create tank/dir1
   sudo zfs create tank/dir2
   ```
   
   * отобразим конструкцию и состояние пула

   ```shell
   zpool status
   ```
   
   * отобразим список датасетов

   ```shell
   zfs list
   ```
   
   * отобразим примонтированные ФС

   ```shell
   df -h
   ```
   
   * создадим внутри пула и датасетов пустые файлы (для наглядности)

   ```shell
   sudo touch ${HOME}/tank/tankfile
   sudo touch ${HOME}/tank/dir1/dir1file
   sudo touch ${HOME}/tank/dir2/dir2file
   ```
   
   * отобразим в виде дерева (утилиту установите сами, если ещё нет)

   ```shell
   tree ${HOME}/tank
   ```

1. Удалим следы экспериментов:

   * удалим пул, датасеты внутри удалятся вместе с пулом
   
   ```shell
   sudo zpool destroy tank
   ```
   
   * удалим файлы
   
   ```shell
   rm ./file{1,2}
   ```

