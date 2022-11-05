### Домашнее задание №16 (SELinux)
#### 1. Создал Vagrantfile согласно представленному в методичке по домашнему заданию. В процессе создания ВМ selinux появилась ошибка SELinux, не позволяющая перевести работу nginx на нестандартный порт.
```console
    selinux: sed: no input files
    selinux: /tmp/vagrant-shell: line 8: /etc/nginx/nginx.conf: Permission denied
    selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Дополнительно установлены пакеты: policycoreutils-python (audit2allow, audit2why).
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
#### 3. Проверил статус SELinux:
```console
[root@selinux ~]# getenforce
Enforcing
```
#### 4. Разврешаем nginx работать на нестандартном порту:
- __Способ 1.__ 
1. В лог-файле __audit.log__ нашел ошибку о блокировании порта 4881:
```console
[root@selinux ~]# less /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1667647551.752:810): avc:  denied  { name_bind } for  pid=2832 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
2. При помощи утилиты audit2why нашел причину блокировки:
  ```console
  [root@selinux ~]# grep 1667647551.752:810 /var/log/audit/audit.log | audit2why 
type=AVC msg=audit(1667647551.752:810): avc:  denied  { name_bind } for  pid=2832 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly. 
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
  ```
3. Включил параметр nis_enabled, рестартанул nginx, проверил статус:
  ```console
  [root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-11-05 14:27:51 UTC; 6s ago
  Process: 21787 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21785 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21784 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21789 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21789 nginx: master process /usr/sbin/nginx
           └─21791 nginx: worker process

Nov 05 14:27:51 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 05 14:27:51 selinux nginx[21785]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 05 14:27:51 selinux nginx[21785]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 05 14:27:51 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
  ```
4. Возвратил настройки обратно. Служба nginx не запускается.
- __Способ 2.__ Разрешим работу порта 4881 с помощью добавления нестандартного порта в имеющийся тип:
  - Найдем имеющийся тип трафика:
  
```console
  [root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

  - Добавил порт 4881 в тип, перезапустил nginx:

  ```console
  [root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881

  [root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-11-05 14:50:53 UTC; 6s ago
  Process: 21857 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21855 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21854 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21859 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21859 nginx: master process /usr/sbin/nginx
           └─21861 nginx: worker process

Nov 05 14:50:53 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 05 14:50:53 selinux nginx[21855]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 05 14:50:53 selinux nginx[21855]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 05 14:50:53 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
  ```
  - Вернул настройки в дефолтное состояние.

- __Способ 3.__ Формируем и устанавливаем модуль SELinux.
  - Посмотрим логи SELinux, связанные с nginx:
  - Воспользуемся утилитой audit2allow, перенаправим логи в эту утилиту:
```console
  root@selinux ~]# cat /var/log/audit/audit.log | grep nginx | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
- Применим данную команду, перезапустим nginx:
```console
root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-11-05 15:14:35 UTC; 7s ago
  Process: 22243 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22241 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22240 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22245 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22245 nginx: master process /usr/sbin/nginx
           └─22247 nginx: worker process

Nov 05 15:14:35 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 05 15:14:35 selinux nginx[22241]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 05 15:14:35 selinux nginx[22241]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 05 15:14:35 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
  - Удалил модуль, nginx не запускается.
____

#### 5. Обеспечение работоспособности приложения при включенном SELinux. 
Создал виртуальные машины ns01 и client, склонированные с репозитория.
#### 6. Попробовал внести изменения в зону, получил ошибку:
```console
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key 
> 192.168.50.10
incorrect section name: 192.168.50.10
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
#### 7. В логах  audit на ВМ client отсутствуют, залогинился на ВМ ns01:
```console
[root@ns01 ~]# cat /var/log/audit/audit.log | grep dns
type=AVC msg=audit(1667662732.397:1902): avc:  denied  { create } for  pid=5095 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
```
#### 8. Убедился, что контексты безопасности не совпадают:
```console
 [root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
#### 9. Поменял контекст безопасности на корректный:
```console
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
```
#### 10. Убедился, что контекст безопасности корректный:
```console
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
#### 11. Обновление зоны прошло успешно:
```console
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key 
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8855
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Nov 05 16:12:08 UTC 2022
;; MSG 
```
#### 12. Перезапустил, настройки сохранились.
