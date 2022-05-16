# NFS, FUSE
1. Подготовка Vagrantfile
2. Настройка сервера и клиента
3. Проверка работоспособности share

#### Подготовка Vagrantfile

Добавим автоматическую установку nfs-utils

```
   Vagrant.configure(2) do |config|
    config.vm.box = "centos/7"
    config.vm.box_version = "2004.01"
    
    config.vm.provider "virtualbox" do |v|
     v.memory = 256
     v.cpus = 1
  end
    config.vm.define "nfss" do |nfss|
     nfss.vm.network "private_network", ip: "192.168.56.10",
    virtualbox__intnet: "net1"
      nfss.vm.hostname = "nfss"
  end
  
    config.vm.define "nfsc" do |nfsc|
     nfsc.vm.network "private_network", ip: "192.168.56.11",
    virtualbox__intnet: "net1"
      nfsc.vm.hostname = "nfsc"
  end
  
  config.vm.provision "shell", inline: <<-SHELL
     sudo yum update
     sudo yum upgrade
     sudo yum install nfs-utils
   SHELL

 end
```

#### Настройка сервера и клиента

1. Включаем фаервол и разрешаем доступ к NFS командами:
```
   systemctl enable firewalld --now
   [vagrant@nfss ~]$ systemctl status firewalld
    ● firewalld.service - firewalld - dynamic firewall daemon
      Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
      Active: active (running) since Mon 2022-05-16 06:07:55 UTC; 1h 1min ago
        Docs: man:firewalld(1)
    Main PID: 404 (firewalld)
      CGroup: /system.slice/firewalld.service
              └─404 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid
  systemctl enable firewalld --now
  firewall-cmd --add-service="nfs3"
  firewall-cmd --add-service="rpc-bind"
  firewall-cmd --add-service="mountd"
  firewall-cmd --permanent --add-port=2049/tcp --add-port=2049/udp --add-port=20048/tcp --add-port=20048/udp --add-port=111/tcp --add-port=111/udp
  firewall-cmd --reload
```
  Включаем сервер NFS, проверяем наличие слушаемых портов
  <<не стал переносить весь ввывод, только по интересующим нас портам>>
```
   systemctl enable nfs --now
   ss -tnplu
   udp    UNCONN     0      0                          *:111                               *:*                   users:(("rpcbind",pid=380,fd=6))
udp    UNCONN     0      0                          *:2049                                   *:*                  
udp    UNCONN     0      0                          *:20048                                  *:*                   users:(("rpc.mountd",pid=796,fd=7))
udp    UNCONN     0      0                       [::]:111                                 [::]:*                   users:(("rpcbind",pid=380,fd=9))
udp    UNCONN     0      0                       [::]:2049                                [::]:*                  
udp    UNCONN     0      0                       [::]:20048                               [::]:*                   users:(("rpc.mountd",pid=796,fd=9))
tcp    LISTEN     0      128                        *:111                                    *:*                   users:(("rpcbind",pid=380,fd=8))
tcp    LISTEN     0      128                        *:20048                                  *:*                   users:(("rpc.mountd",pid=796,fd=8))
tcp    LISTEN     0      64                         *:2049                                   *:*                  
tcp    LISTEN     0      128                     [::]:111                                 [::]:*                   users:(("rpcbind",pid=380,fd=11))
tcp    LISTEN     0      128                     [::]:20048                             [::]:*                   users:(("rpc.mountd",pid=796,fd=10))
tcp    LISTEN     0      64                      [::]:2049                                [::]:*                  
```

  Создаем и настраиваем директорию для share
```
   mkdir -p /srv/share/upload
   chown -R nfsnobody:nfsnobody /srv/share
   chmod 0777 /srv/share/upload
```

  Создаем структуру, которая позволит экспортировать ранее созданную директорию
```
   cat << EOF > /etc/exports
   /srv/share 192.168.50.11/32(rw,sync,root_squash)
   EOF
   exportfs -r
```

  Проверяем директорию
```
   [root@nfss vagrant]# exportfs -s
   /srv/share  192.168.56.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
   /srv/share  192.168.56.11/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```
 
  Подключаемся к клиенту и производим настройки
```
   systemctl enable firewalld --now
   [root@nfsc vagrant]# systemctl status firewalld
    ● firewalld.service - firewalld - dynamic firewall daemon
      Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
      Active: active (running) since Mon 2022-05-16 06:08:16 UTC; 1h 30min ago
        Docs: man:firewalld(1)
    Main PID: 401 (firewalld)
      CGroup: /system.slice/firewalld.service
              └─401 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

   May 16 06:08:15 nfsc systemd[1]: Starting firewalld - dynamic firewall daemon...
   May 16 06:08:16 nfsc systemd[1]: Started firewalld - dynamic firewall daemon.
   May 16 06:08:16 nfsc firewalld[401]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure... now.
   Hint: Some lines were ellipsized, use -l to show in full.
```
   
  Добовляем в /etc/fstab строку
```
   echo "192.168.50.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
   systemctl daemon-reload
   systemctl restart remote-fs.target
```
  
  Заходим и проверяем директорию на монтирование
```
   [root@nfsc vagrant]# cd /mnt/
   [root@nfsc mnt]# mount | grep mnt
   systemd-1 on /mnt type autofs (rw,relatime,fd=26,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=10869)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
```

#### Проверка работоспособности

  Заходим на сервер и создаем файл (т.к. ранее я уже все это создовал, но при пуше на git удалил всю документацию, удалим все ранее созданые файлы и создадим новые) 
```
   [root@nfss upload]# touch newwork_file
   [root@nfss upload]# ll
   total 0
   -rw-r--r--. 1 root root 0 May 16 08:09 newwork_file
```

  Заходим на клиент проверяем и тоже создадим файл
```
   [root@nfsc upload]# ls -la
   total 0
   drwxrwxrwx. 2 nfsnobody nfsnobody 26 May 16 08:09 .
   drwxr-xr-x. 3 nfsnobody nfsnobody 20 Apr 25 06:55 ..
   -rw-r--r--. 1 root      root       0 May 16 08:09 newwork_file
   [root@nfsc upload]# touch soon_change
   [root@nfsc upload]# ls -la
   total 0
   drwxrwxrwx. 2 nfsnobody nfsnobody 45 May 16 08:12 .
   drwxr-xr-x. 3 nfsnobody nfsnobody 20 Apr 25 06:55 ..
   -rw-r--r--. 1 root      root       0 May 16 08:09 newwork_file
   -rw-r--r--. 1 nfsnobody nfsnobody  0 May 16 08:12 soon_change
```

  Проверяе сохраняются ли настройки, перезагрузим клиент
```
   [root@nfsc upload]# cd ~
   [root@nfsc ~]# reboot
   Connection to 127.0.0.1 closed by remote host.
   satimur@asus-tuf-dash-f15:~/otus/git/homework5$ vagrant ssh nfsc
   Last login: Mon May 16 07:02:07 2022 from 10.0.2.2
   [vagrant@nfsc ~]$ sudo su
   [root@nfsc vagrant]# cd /mnt/
   [root@nfsc mnt]# ll
   total 0
   drwxrwxrwx. 2 nfsnobody nfsnobody 45 May 16 08:12 upload
   [root@nfsc mnt]# cd upload
   [root@nfsc upload]# ll
   total 0
   -rw-r--r--. 1 root      root      0 May 16 08:09 newwork_file
   -rw-r--r--. 1 nfsnobody nfsnobody 0 May 16 08:12 soon_change
```
 
  Проверяем сервер
```
   [root@nfss upload]# cd ~
   [root@nfss ~]# reboot
   Connection to 127.0.0.1 closed by remote host.
   satimur@asus-tuf-dash-f15:~/otus/git/homework5$ vagrant ssh nfss
   Last login: Mon May 16 07:01:52 2022 from 10.0.2.2
   [vagrant@nfss ~]$ sudo su
   [root@nfss vagrant]# cd /srv/share/upload
   [root@nfss upload]# ll
   total 0
   -rw-r--r--. 1 root      root      0 May 16 08:09 newwork_file
   -rw-r--r--. 1 nfsnobody nfsnobody 0 May 16 08:12 soon_change
   [root@nfss upload]# systemctl status nfs
    ● nfs-server.service - NFS server and services
      Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
     Drop-In: /run/systemd/generator/nfs-server.service.d
              └─order-with-mounts.conf
      Active: active (exited) since Mon 2022-05-16 08:20:49 UTC; 1h 46min ago
     Process: 830 ExecStartPost=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gssproxy ; fi (code=exited, status=0/SUCCESS)
     Process: 800 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
     Process: 798 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Main PID: 800 (code=exited, status=0/SUCCESS)
      CGroup: /system.slice/nfs-server.service

   May 16 08:20:49 nfss systemd[1]: Starting NFS server and services...
   May 16 08:20:49 nfss systemd[1]: Started NFS server and services.
   [root@nfss upload]# systemctl status firewalld
    ● firewalld.service - firewalld - dynamic firewall daemon
      Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
      Active: active (running) since Mon 2022-05-16 08:20:47 UTC; 1h 46min ago
        Docs: man:firewalld(1)
    Main PID: 405 (firewalld)
      CGroup: /system.slice/firewalld.service
              └─405 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

   May 16 08:20:46 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
   May 16 08:20:47 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
   May 16 08:20:47 nfss firewalld[405]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure... now.
   Hint: Some lines were ellipsized, use -l to show in full.
   [root@nfss upload]# exportfs -s
   /srv/share  192.168.56.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
   /srv/share  192.168.56.11/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
   [root@nfss upload]# showmount -a 192.168.56.10
   All mount points on 192.168.56.10:
   192.168.56.11:/srv/share
```

  Возвращаемся на клиент и перезагружаем его, проверяем и создаем файл
```
   [root@nfsc upload]# cd ~
   [root@nfsc ~]# reboot 
   Connection to 127.0.0.1 closed by remote host.
   satimur@asus-tuf-dash-f15:~/otus/git/homework5$ vagrant ssh nfsc
   Last login: Mon May 16 10:00:06 2022 from 10.0.2.2
   [vagrant@nfsc ~]$ sudo su
   [root@nfsc vagrant]# ll
   total 0
   [root@nfsc vagrant]#     
   [root@nfsc vagrant]# 
   [root@nfsc vagrant]# showmount -a 192.168.56.10
   All mount points on 192.168.56.10:
   192.168.56.11:/srv/share
   [root@nfsc vagrant]# cd /mnt/upload
   [root@nfsc upload]# ll
   total 0
   -rw-r--r--. 1 root      root      0 May 16 08:09 newwork_file
   -rw-r--r--. 1 nfsnobody nfsnobody 0 May 16 08:12 soon_change
   [root@nfsc upload]# mount |grep mnt
   systemd-1 on /mnt type autofs (rw,relatime,fd=32,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=11149)
   192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
   [root@nfsc upload]# mount | grep mnt
   systemd-1 on /mnt type autofs (rw,relatime,fd=32,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=11149)
   192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
   [root@nfsc upload]# touch exactly_this_file
   [root@nfsc upload]# ls -la
   total 0
   drwxrwxrwx. 2 nfsnobody nfsnobody 70 May 16 10:19 .
   drwxr-xr-x. 3 nfsnobody nfsnobody 20 Apr 25 06:55 ..
   -rw-r--r--. 1 nfsnobody nfsnobody  0 May 16 10:19 exactly_this_file
   -rw-r--r--. 1 root      root       0 May 16 08:09 newwork_file
   -rw-r--r--. 1 nfsnobody nfsnobody  0 May 16 08:12 soon_change
   [root@nfsc upload]# 
```

  Проверим также и на сервере, если там ранее созданый нами файл
```
   [root@nfss upload]# ls -la
   total 0
   drwxrwxrwx. 2 nfsnobody nfsnobody 70 May 16 10:19 .
   drwxr-xr-x. 3 nfsnobody nfsnobody 20 Apr 25 06:55 ..
   -rw-r--r--. 1 nfsnobody nfsnobody  0 May 16 10:19 exactly_this_file
   -rw-r--r--. 1 root      root       0 May 16 08:09 newwork_file
   -rw-r--r--. 1 nfsnobody nfsnobody  0 May 16 08:12 soon_change
```
   


  
  
  
  
