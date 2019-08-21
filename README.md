## Уменьшить том под / до 8G
Так как файловая система xfs не поддерживает изменения в сторону  уменьшения своего размера, то необходимо использовать вспомогательную директорию в которую будет временно
скопирован контент из корневой директории затем удален и создан заново корневой раздел под управлением xfs размером 8G, куда обратно будет восстановлен контент из временной директории.
Для этого устанавливаем утилиту xfsdump. Выбираем в качестве временного хранилища диск /dev/sdb.

	pvcreate /dev/sdb  --создаем физический раздел 
	vgcreate vg_root /dev/sdb -- создаем группу
	lvcreate -n lv_root -l 100%FREE /dev/vg_root  -- создаем логический раздел в ранее созданной группе
	mkfs.xfs /dev/vg_root/lv_root -- созадем файловую систему xfs в логическом разделе
	mount /dev/vg_root/lv_root /mnt -- монтируем  раздел в точку монтирования /mnt 
	xfsdump -J - /VolGroup00/LogVol00 | xfsrestore -J - /mnt --  снимаем дамп с корневого раздела и восстанавливаем его в /mnt
	
Далее, необходимо переконфигурировать загрузчик, чтобы загрузка шла из новой директории. 


	for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done -- Создаем синонимы корневых директорий в директорию /mnt
	chroot /mnt  --изменяем местоположение корневого каталога на /mnt
	grub2-mkconfig -o /boot/grub2/grub.cfg -- создаем конфигурационный файл загрузчика  и редактируем в нем rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root
	cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done -- обновляем образ ядра 
	
После перезагрузки системы убеждаемся что корневой раздел переехал в новое местоположение

	[root@lvm vagrant]# lsblk
	NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sda                       8:0    0   40G  0 disk
	├─sda1                    8:1    0    1M  0 part
	├─sda2                    8:2    0    1G  0 part /boot
	└─sda3                    8:3    0   39G  0 part
	  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
	  └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm
	sdb                       8:16   0    8G  0 disk
	└─vg_root-lv_root       253:0    0    8G  0 lvm  /
	sdc                       8:32   0    8G  0 disk
	sdd                       8:48   0    8G  0 disk

Удаляем старый раздел 

		 lvremove /dev/VolGroup00/LogVol00
		 
Создаем новый раздел размером в нужные 8Гб и создаем на нем файловую системы xfs, монтируем раздел в /mnt

	lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
	mkfs.xfs /dev/VolGroup00/LogVol00
	mount /dev/VolGroup00/LogVol00 /mnt
	xfsdump -J - /vg_root/lv_root | xfsrestore -J - /mnt  -- снимаем дамп с текущего корневого раздела и восстанавливаем его в /mnt
	
Далее, аналогичным образом переконфигурируем загрузчик для работы из новой директории 

	for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done -- Создаем синонимы корневых директорий в директорию /mnt
	chroot /mnt  --изменяем местоположение корневого каталога на /mnt
	grub2-mkconfig -o /boot/grub2/grub.cfg -- создаем конфигурационный файл загрузчика
	cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done -- обновляем образ ядра 
***
## Создаем зеркало для /var
Оставаясь в /mnt в качестве корневого каталога (из предыдущего пункта)  создадим зеркальный раздел на свободных дисках

	pvcreate /dev/sdb /dev/sdc -- создаем физический раздел
	vgcreate vg_var /dev/sdc /dev/sdd  -- создаем группу 
	lvcreate -L 950M -m1 -n lv_var vg_var -- создаем  логический раздел в зеркале
	mkfs.ext4 /dev/vg_var/lv_var  -- создаем файловую систему ext4 в разделе 
	mount /dev/vg_var/lv_var /mnt -- монтируем в /mnt
	rsync -avHPSAX /var/ /mnt/ -- копируем и синхронизируем данные из /var/ в новую директорию
	umount /mnt 
	mount /dev_vg_var/lv_var /var -- перемонтируем в новый каталог /var
	
Для монтирования при загрузке  прописываем в /etc/fstab 

	echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

После перезагрузки

	[root@lvm vagrant]# lsblk
	NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sda                        8:0    0   40G  0 disk
	├─sda1                     8:1    0    1M  0 part
	├─sda2                     8:2    0    1G  0 part /boot
	└─sda3                     8:3    0   39G  0 part
		├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
		└─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
	sdb                        8:16   0    8G  0 disk
	└─vg_root-lv_root        253:7    0    8G  0 lvm
	sdc                        8:32   0    8G  0 disk
	├─vg_var-lv_var_rmeta_0  253:2    0    4M  0 lvm
	│ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
	└─vg_var-lv_var_rimage_0 253:3    0  952M  0 lvm
	  └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
	sdd                        8:48   0    8G  0 disk
	├─vg_var-lv_var_rmeta_1  253:4    0    4M  0 lvm
	│ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
	└─vg_var-lv_var_rimage_1 253:5    0  952M  0 lvm
	  └─vg_var-lv_var        253:6    0  952M  0 lvm  /var


Теперь  можно удалить временный раздел 
		
		 lvremove /dev/vg_root/lv_root
		 vgremove /dev/vg_root
		 pvremove /dev/sdb
		 
***
## Выделение тома под /home и создание снэпшота
Создадим том в существующей группе VolGroup00.

	 lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
	 mkfs.xfs /dev/VolGroup00/LogVol_Home
	 mount /dev/VolGroup00/LogVol_Home /mnt/
	 rsync -avHPSAX /home/ /mnt/
	 rm -rf /home/*
	 umount /mnt
	 mount /dev/VolGroup00/LogVol_Home /home 
	 echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
	 

	[root@lvm vagrant]# lsblk
	NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sda                          8:0    0   40G  0 disk
	├─sda1                       8:1    0    1M  0 part
	├─sda2                       8:2    0    1G  0 part /boot
	└─sda3                       8:3    0   39G  0 part
	  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
	  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
	  └─VolGroup00-LogVol_Home 253:7    0    2G  0 lvm  /home
	sdb                          8:16   0    8G  0 disk
	sdc                          8:32   0    8G  0 disk
	├─vg_var-lv_var_rmeta_0    253:2    0    4M  0 lvm
	│ └─vg_var-lv_var          253:6    0  952M  0 lvm  /var
	└─vg_var-lv_var_rimage_0   253:3    0  952M  0 lvm
	└─vg_var-lv_var          253:6    0  952M  0 lvm  /var
	sdd                          8:48   0    8G  0 disk
	├─vg_var-lv_var_rmeta_1    253:4    0    4M  0 lvm
	│ └─vg_var-lv_var          253:6    0  952M  0 lvm  /var
	└─vg_var-lv_var_rimage_1   253:5    0  952M  0 lvm
	└─vg_var-lv_var          253:6    0  952M  0 lvm  /var

	
Сгенерируем файлы в /home 

	[root@lvm vagrant]# for i in `seq 1 10`
	> do
	> touch /home/file$i
	> done
	[root@lvm vagrant]# ls /home
	file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  vagrant
	[root@lvm vagrant]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home   -- создали снэпшот home_snap 
	Rounding up size to full physical extent 128.00 MiB
	Logical volume "home_snap" created.
	[root@lvm vagrant]# rm -f /home/file{1..5}    -- удалили несколько файлов 
	[root@lvm vagrant]# ls /home
	file10  file6  file7  file8  file9  vagrant
	[root@lvm vagrant]# umount /home
	[root@lvm vagrant]# lvconvert --merge /dev/VolGroup00/home_snap   -- провели восстановление из снэпшота
	Merging of volume VolGroup00/home_snap started.
	VolGroup00/LogVol_Home: Merged: 100.00%
	[root@lvm vagrant]# mount /home 
	[root@lvm vagrant]# ls /home
	file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  vagrant
***	
##Установка zfs и перенос каталога /opt
Устанавливаем репозитарий zfs так как эта файловая система не входит в дистрибутив centOS.

	sudo yum install http://download.zfsonlinux.org/epel/zfs-release.el7_5.noarch.rpm
	
Редактируем файл /etc/yum.repos.d/zfs.repo  для выбора репозитария KMOD.

Устанавливаем zfs  

	yum install zfs
	reboot
	modprobe zfs  -- загружаем вручную модуль, так как автоматически он не подргрузился после перезагрузки 
	[root@lvm vagrant]# lsmod | grep zfs
	zfs                  3564468  4
	zunicode              331170  1 zfs
	zavl                   15236  1 zfs
	icp                   270148  1 zfs
	zcommon                73440  1 zfs
	znvpair                89131  2 zfs,zcommon
	spl                   102412  4 icp,zfs,zcommon,z	nvpair
	

Создадим пул данных на двух дисках

	zpool create zfspool /dev/sde /dev/sdf

В новом пуле создадим датасет
	mkdir /opt1
	zpool create zfspool/opt -o mountpoint=/opt1

Создадим снэпшот для нового датасета

	zfs snapshot zfspool/opt@version1
	
	zfs list -t snapshot 
	[root@lvm vagrant]# zfs list -t snapshot
	NAME                   USED  AVAIL  REFER  MOUNTPOINT
	zfspool/opt@version1    22K      -    24K  -
	
Создадим кэш на отдельном диске 

	zpool add zfspool log /dev/sdg
	
	[root@lvm vagrant]# zpool status zfspool
	pool: zfspool
	state: ONLINE
	scan: none requested
	config:

        NAME        STATE     READ WRITE CKSUM
        zfspool     ONLINE       0     0     0
          sde       ONLINE       0     0     0
          sdf       ONLINE       0     0     0
        logs
          sdg       ONLINE       0     0     0

	errors: No known data errors
	 touch /opt/file{1..20}
	 rsync -avHPSAX /opt/ /opt1/
	 rm -rf /opt/*
	 umount /opt1
	 zfs set mountpoint=/opt  zfspool/opt
	 
	 [root@lvm vagrant]# zfs get mountpoint zfspool/opt
	NAME         PROPERTY    VALUE       SOURCE
	zfspool/opt  mountpoint  /opt        local
	 
	[root@lvm vagrant]# ls /opt
	file1   file11  file13  file15  file17  file19  file20  file4  file6  file8
	file10  file12  file14  file16  file18  file2   file3   file5  file7  file9 
