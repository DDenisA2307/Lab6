```
Домашнее задание 6

Работа с NFS

1. Настраиваем nfs сервер

denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant ssh nfss
[vagrant@nfss ~]$ sudo -i
[root@nfss ~]# yum install nfs-utils
...
Updated:
  nfs-utils.x86_64 1:1.3.0-0.68.el7.2                                                                   

Complete!
[root@nfss ~]# systemctl enable firewalld --now 
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.
[root@nfss ~]# firewall-cmd --add-service="nfs3" \
> --add-service="rpc-bind" \
> --add-service="mountd" \
> --permanent 
success
[root@nfss ~]# firewall-cmd --reload
success
[root@nfss ~]# systemctl enable nfs --now 
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.
[root@nfss ~]# mkdir -p /srv/share/upload 
[root@nfss ~]# chown -R nfsnobody:nfsnobody /srv/share 
[root@nfss ~]# chmod 0777 /srv/share/upload
[root@nfss ~]# cat << EOF > /etc/exports 
> /srv/share 192.168.56.11/32(rw,sync,root_squash)
> EOF
[root@nfss ~]# exportfs -r 
[root@nfss ~]# exportfs -s
/srv/share  192.168.56.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
[root@nfss ~]# exit
logout
[vagrant@nfss ~]$ exit
logout

2. Настраиваем клиент nfs

denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant ssh nfsc
[vagrant@nfsc ~]$ sudo -i
[root@nfsc ~]# yum install nfs-utils
...
Updated:
  nfs-utils.x86_64 1:1.3.0-0.68.el7.2                                                                   

Complete!
[root@nfsc ~]# echo "192.168.56.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
[root@nfsc ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Thu Apr 30 22:04:55 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15 /                       xfs     defaults        0 0
/swapfile none swap defaults 0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
192.168.56.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0
[root@nfsc ~]# systemctl daemon-reload 
[root@nfsc ~]# systemctl restart remote-fs.target 
[root@nfsc ~]# mount | grep mnt 
systemd-1 on /mnt type autofs (rw,relatime,fd=24,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=27089)
[root@nfsc ~]# cd /mnt/
[root@nfsc mnt]# mount | grep mnt 
systemd-1 on /mnt type autofs (rw,relatime,fd=24,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=27089)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)


3. Проверяем работоспособность

создаем файл на сервере:

denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant ssh nfss
Last login: Thu Dec  7 06:32:21 2023 from 10.0.2.2
[vagrant@nfss ~]$ cd /srv/share/upload
[vagrant@nfss upload]$ touch check_file

Проверяем на клиенте:

[root@nfsc mnt]# cd /mnt/upload
[root@nfsc upload]# ls
check_file

Создаем файл на клиенте:

[root@nfsc upload]# touch client_file
[root@nfsc upload]# ls
check_file  client_file

Проверяем на сервере:

[vagrant@nfss upload]$ ls
check_file  client_file

4. Проверка после перезагрузки:

Перезагружаем клиент:

[root@nfsc upload]# exit
logout
[vagrant@nfsc ~]$ exit
logout
denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant reload nfsc
denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant ssh nfsc
Last login: Thu Dec  7 06:48:40 2023 from 10.0.2.2
[vagrant@nfsc ~]$ cd /mnt/upload
[vagrant@nfsc upload]$ ls
check_file  client_file

Перезагружаем сервер:

denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant reload nfss
denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant ssh nfss
Last login: Thu Dec  7 07:03:08 2023 from 10.0.2.2
[vagrant@nfss ~]$ cd /srv/share/upload/
[vagrant@nfss upload]$ ls
check_file  client_file
[vagrant@nfss upload]$ systemctl status nfs
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: active (exited) since Thu 2023-12-07 07:20:33 UTC; 2min 31s ago
  Process: 809 ExecStartPost=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gssproxy ; fi (code=exited, status=0/SUCCESS)
  Process: 789 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 782 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 789 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service
[vagrant@nfss upload]$ systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2023-12-07 07:20:21 UTC; 3min 7s ago
     Docs: man:firewalld(1)
 Main PID: 407 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─407 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid
[vagrant@nfss upload]$ exportfs -s
exportfs: could not open /var/lib/nfs/.etab.lock for locking: errno 13 (Permission denied)
[vagrant@nfss upload]$ sudo exportfs -s 
/srv/share  192.168.56.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
[vagrant@nfss upload]$ showmount -a 192.168.56.10
All mount points on 192.168.56.10:
192.168.56.11:/srv/share

Перезагружаем клиент:

denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant reload nfsc
denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant ssh nfsc
Last login: Thu Dec  7 07:28:37 2023 from 10.0.2.2
[vagrant@nfsc ~]$ sudo -i
[root@nfsc ~]# showmount -a 192.168.56.10
All mount points on 192.168.56.10:
192.168.56.11:/srv/share
[root@nfsc ~]# cd /mnt/upload
[root@nfsc upload]# ls
check_file  client_file
[root@nfsc upload]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=33,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=11207)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
[root@nfsc upload]# touch final_check
[root@nfsc upload]# ls
check_file  client_file  final_check

В гит будет загружено два файла Vagrantfile^
1. Без автоматического создания при поднятии ВМ
2. с автоматическим созданием nfs

Проверка со скриптами:

Сервер:

denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant ssh nfss
[vagrant@nfss ~]$ sudo -i
[root@nfss ~]# exportfs -s 
/srv/share  192.168.56.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
[root@nfss ~]# cd /srv/share/upload/
[root@nfss upload]# systemctl status nfs
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
   Active: active (exited) since Thu 2023-12-07 10:23:08 UTC; 7min ago
  Process: 3674 ExecStartPost=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gssproxy ; fi (code=exited, status=0/SUCCESS)
  Process: 3657 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 3656 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 3657 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Dec 07 10:23:07 nfss systemd[1]: Starting NFS server and services...
Dec 07 10:23:08 nfss systemd[1]: Started NFS server and services.
[root@nfss upload]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2023-12-07 10:23:03 UTC; 7min ago
     Docs: man:firewalld(1)
 Main PID: 3498 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─3498 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Dec 07 10:23:02 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
Dec 07 10:23:03 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
Dec 07 10:23:03 nfss firewalld[3498]: WARNING: AllowZoneDrifting is enabled...w.
Dec 07 10:23:06 nfss firewalld[3498]: WARNING: AllowZoneDrifting is enabled...w.
Hint: Some lines were ellipsized, use -l to show in full.
[root@nfss upload]# showmount -a 192.168.56.10
All mount points on 192.168.56.10:
192.168.56.11:/srv/share

Клиент:

denis@DDA-VirtualBox:~/Lab6_nfs$ vagrant ssh nfsc
[vagrant@nfsc ~]$ sudo -i
[root@nfsc ~]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=46,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=26177)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
[root@nfsc ~]# cd /mnt
[root@nfsc mnt]# ls
upload
[root@nfsc mnt]# showmount -a 192.168.56.10
All mount points on 192.168.56.10:
192.168.56.11:/srv/share
[root@nfsc mnt]# cd /mnt/upload
[root@nfsc upload]# ls
[root@nfsc upload]# touch final_check
[root@nfsc upload]# ls
final_check

