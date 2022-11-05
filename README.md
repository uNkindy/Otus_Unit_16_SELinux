### Домашнее задание №16 (SELinux)
#### 1. Создал Vagrantfile согласно представленному в методичке по домашнему заданию. В процессе создания ВМ selinux появилась ошибка SELinux, не позволяющая перевести работу nginx на нестандартный порт.
```console
    selinux: sed: no input files
    selinux: /tmp/vagrant-shell: line 8: /etc/nginx/nginx.conf: Permission denied
    selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
#### 2. Проверил отключенный firewalld и корректность конфигов nginx:
```console
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)

[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
#### 3. 