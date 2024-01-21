## Стенд с Vagrant c SELinux

### Цели домашнего задания

Диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.

### Описание домашнего задания

1. Запустить nginx на нестандартном порту 3-мя разными способами:
   - переключатели setsebool;
   - добавление нестандартного порта в имеющийся тип;
   - формирование и установка модуля SELinux.
   К сдаче:README с описанием каждого решения (скриншоты и демонстрация приветствуются). 

2. Обеспечить работоспособность приложения при включенном selinux.
  - развернуть приложенный [стенд](https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems); 
  - выяснить причину неработоспособности механизма обновления зоны (см. README);
  - предложить решение (или решения) для данной проблемы;
  - выбрать одно из решений для реализации, предварительно обосновав выбор;
  - реализовать выбранное решение и продемонстрировать его работоспособность.


### Краткая информация

SElinux (Security Enhanced Linux) – система принудительного (мандатного) контроля доступа (MAC). Разрабатывалась АНБ. В 2003 году вошла в состав ядра linux 2.6.x.

SELinux следует модели минимально необходимых привилегий для каждого сервиса пользователя и программы.

Зачем нужен Selinux:

- Гибкое ограничение прав пользователей и процессов на уровне ядра
- Работа совместно с DAC (матричным управлением доступа)
- Снижение риска, возникающего вследствие допущенных ошибок
- Ограничение потенциально опасных или скомпрометированных процессов в правах
- Протоколирование

Обычно в В Selinux большие и сложные политики. Каждый ресурс должен быть описан и сопоставлен с сервисом.

Режимы работы SELinux:

- Enforcing (по-умолчанию) – Активная работа. Всё, что нарушает политику безопасности блокируется. Попытка нарушения фиксируется в журнале
- Permissive — запрещенные действия не блокируются. Все нарушения пишутся в журнал
- Disabled — полное отключение SELinux.

> Важно помнить: классическая система прав Unix применяется первой и управление перейдёт к SELinux только в том случае, если эта первичная проверка будет успешно пройдена

Написать политику SELinux достаточно сложно, но для ключевых приложений и сервисов (например httpd, mysqld, dhcpd и т.д.) определены заранее сконфигурированные политики, которые не позволят получить злоумышленнику доступ к важным данным.  
Те приложения, для которых политика не определена, выполняются в домене unconfined_f и не защищаются SELinux.
>- SELinux Для пользователей добавляется суффикс «_u», для ролей - «_r», а для типов (для файлов) или доменов (для процессов) - «_t».

### Предварительные условия 
Требуется предварительно установленный и работоспособный [Hashicorp Vagrant](https://www.vagrantup.com/downloads) и [Oracle VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads). 

### Создание виртуальной машины

- Создать [Vagrantfile]() и выполнить vagrant up для создания виртуальной машины с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.

Во время развёртывания стенда попытка запустить nginx завершается с ошибкой

```bash
selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Thu 2024-01-11 04:55:34 UTC; 7ms ago
    selinux:   Process: 2885 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 2884 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux: 
    selinux: Jan 11 04:55:34 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Jan 11 04:55:34 selinux nginx[2885]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Jan 11 04:55:34 selinux nginx[2885]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: Jan 11 04:55:34 selinux nginx[2885]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Jan 11 04:55:34 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: Jan 11 04:55:34 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: Jan 11 04:55:34 selinux systemd[1]: Unit nginx.service entered failed state.
    selinux: Jan 11 04:55:34 selinux systemd[1]: nginx.service failed.
```

>- Данная ошибка появляется из-за того, что SELinux блокирует работу nginx на нестандартном порту.
>- Для захода на сервер используется команда: vagrant ssh
>- Дальнейшие действия выполняются от пользователя root. Переход в root пользователя: sudo -i

```
elena_leb@ubuntunbleb:~/SELinux_DZ$ vagrant ssh
[vagrant@selinux ~]$ systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2024-01-11 04:55:34 UTC; 3min 9s ago
  Process: 2885 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2884 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
```

### 1. Запуск nginx на нестандартном порту 3-мя разными способами

- Проверить, что в ОС отключен файервол

```
[vagrant@selinux ~]$ sudo -i
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

- Проверить, что конфигурация nginx настроена без ошибок

```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

- Далее проверим режим работы SELinux

```
[root@selinux ~]# getenforce
Enforcing
```
>- Должен отображаться режим Enforcing. Данный режим означает, что SELinux будет блокировать запрещенную активность

### 1 способ - Разрешение в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

- Выполнить поиск в логах (/var/log/audit/audit.log) информации о блокировании порта

```
[root@selinux ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1704948934.287:830): avc:  denied  { name_bind } for  pid=2885 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

- Ориентируясь на время, в которое был записан этот лог, с помощью утилиты audit2why посмотреть информацию о запрете

```
[root@selinux ~]# grep 1704948934.287:830 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1704948934.287:830): avc:  denied  { name_bind } for  pid=2885 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```

>- Утилита audit2why покажет почему трафик блокируется. 

- Вывод утилиты, указывает, что нужно поменять параметр nis_enabled. Включить параметр nis_enabled и перезапустить nginx

```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-01-11 05:16:06 UTC; 16s ago
  Process: 3101 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3099 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3098 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3103 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3103 nginx: master process /usr/sbin/nginx
           └─3105 nginx: worker process

Jan 11 05:16:06 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 11 05:16:06 selinux nginx[3099]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 11 05:16:06 selinux nginx[3099]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 11 05:16:06 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

- Проверка работы nginx из браузера. Заходим в любой браузер на хосте и переходим по адресу <http://127.0.0.1:4881>

![screenshot1](https://github.com/ellopa/otus-selinux/blob/main/SELinux-2024-01-11_1.png)

- Проверить статус параметра можно с помощью команды

```
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```

- Вернуть запрет работы nginx на порту 4881 обратно. Для этого отключить nis_enabled

```bash
setsebool -P nis_enabled off
```

>- После отключения nis_enabled служба nginx снова не запустится
```
[root@selinux ~]# systemctl stop nginx
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2024-01-11 05:24:05 UTC; 18s ago
  Process: 21963 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 21962 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Jan 11 05:24:05 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 11 05:24:05 selinux nginx[21963]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 11 05:24:05 selinux nginx[21963]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 11 05:24:05 selinux nginx[21963]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 11 05:24:05 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jan 11 05:24:05 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jan 11 05:24:05 selinux systemd[1]: Unit nginx.service entered failed state.
Jan 11 05:24:05 selinux systemd[1]: nginx.service failed.
```

### 2 способ - Разрешение в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип

- Найти имеющейся тип, для http трафика
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

- Добавить порт в тип http_port_t
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

- Перезапустить службу nginx и проверить
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-01-11 05:28:52 UTC; 7s ago
  Process: 22013 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22011 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22010 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22015 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22015 nginx: master process /usr/sbin/nginx
           └─22016 nginx: worker process

Jan 11 05:28:52 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 11 05:28:52 selinux nginx[22011]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 11 05:28:52 selinux nginx[22011]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 11 05:28:52 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

- Проверка работы nginx из браузера. Заходим в любой браузер на хосте и переходим по адресу <http://127.0.0.1:4881>

![screenshot2](https://github.com/ellopa/otus-selinux/blob/main/SELinux-2024-01-11_2.png)

- Удалить нестандартный порт из имеющегося типа
```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
```
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2024-01-11 05:33:26 UTC; 37s ago
  Process: 22013 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22061 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 22060 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22015 (code=exited, status=0/SUCCESS)

Jan 11 05:33:26 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 11 05:33:26 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 11 05:33:26 selinux nginx[22061]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 11 05:33:26 selinux nginx[22061]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 11 05:33:26 selinux nginx[22061]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 11 05:33:26 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jan 11 05:33:26 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jan 11 05:33:26 selinux systemd[1]: Unit nginx.service entered failed state.
Jan 11 05:33:26 selinux systemd[1]: nginx.service failed.
```

### 3 способ - Разрешение в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux

- Выполнить запуск nginx
```
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

- Nginx не запустился, так как SELinux продолжает его блокировать. Посмотреть логи SELinux, которые относятся к nginx
```bash
grep nginx /var/log/audit/audit.log
```
```
type=SYSCALL msg=audit(1704948934.287:830): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55906dfc78b8 a2=10 a3=7fff371677a0 items=0 ppid=1 pid=2885 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1704948934.289:831): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
- Запустить утилиту audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
>- Audit2allow формирует модуль и указывает команду, с помощью которой можно применить данный модуль

- Выполнить команду
```bash
semodule -i nginx.pp
```
- Запустить nginx
```
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-01-11 05:41:04 UTC; 11s ago
  Process: 22144 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22142 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22141 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22146 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22146 nginx: master process /usr/sbin/nginx
           └─22148 nginx: worker process

Jan 11 05:41:04 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 11 05:41:04 selinux nginx[22142]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 11 05:41:04 selinux nginx[22142]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 11 05:41:04 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

>- После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки.  
>- Просмотр всех установленных модулей: semodule -l

- Удаление модуля выполняется командой semodule -r nginx
```
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```

### 2. Обеспечение работоспособности приложения при включенном SELinux

### Предварительные условия 
Для того, чтобы развернуть стенд потребуется хост, с установленным git и ansible.
Инструкция по установке [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
Инструкция по установке [Git](https://git-scm.com/book/ru/v2/%D0%92%D0%B2%D0%B5%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5-%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-Git)

- Выполнить клонирование репозитория

```
elena_leb@ubuntunbleb:~/CLONE_SELinux$ git clone https://github.com/mbfx/otus-linux-adm.git
Клонирование в «otus-linux-adm»...
remote: Enumerating objects: 558, done.
remote: Counting objects: 100% (456/456), done.
remote: Compressing objects: 100% (303/303), done.
remote: Total 558 (delta 125), reused 396 (delta 74), pack-reused 102
Получение объектов: 100% (558/558), 1.38 МиБ | 1.57 МиБ/с, готово.
Определение изменений: 100% (140/140), готово.
```

- Перейти в каталог со стендом: cd otus-linux-adm/selinux_dns_problems

- Развернуть 2 ВМ с помощью vagrant: vagrant up
```
elena_leb@ubuntunbleb:~/CLONE_SELinux/otus-linux-adm/selinux_dns_problems$ vagrant up
Bringing machine 'ns01' up with 'virtualbox' provider...
Bringing machine 'client' up with 'virtualbox' provider...
==> ns01: Importing base box 'centos/7'...
==> ns01: Matching MAC address for NAT networking...
==> ns01: Checking if box 'centos/7' version '2004.01' is up to date...
……..
    client: Running ansible-playbook...

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [client]

TASK [install packages] ********************************************************
changed: [client]

PLAY [ns01] ********************************************************************
skipping: no hosts matched

PLAY [client] ******************************************************************

TASK [Gathering Facts] *********************************************************
ok: [client]

TASK [copy resolv.conf to the client] ******************************************
changed: [client]

TASK [copy rndc conf file] *****************************************************
changed: [client]
TASK [copy motd to the client] *************************************************
changed: [client]

TASK [copy transferkey to client] **********************************************
changed: [client]

PLAY RECAP *********************************************************************
client                     : ok=7    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```
- Проверить ВМ с помощью команды: vagrant status
```
elena_leb@ubuntunbleb:~/CLONE_SELinux/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

```

- Подключиться к клиенту: vagrant ssh client
```
elena_leb@ubuntunbleb:~/CLONE_SELinux/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
Last login: Thu Jan 11 06:46:51 2024 from 10.0.2.2
###################################
### Welcome to the DNS lab! ###
###################################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###################################
### Enjoy! #######################
###################################
[vagrant@client ~]$ 
```
- Внести изменения в зону

>- [nsupdate: динамический редактор зон DNS](https://yamadharma.github.io/ru/post/2023/10/28/nsupdate-dynamic-dns-editor/)

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```

- Изменения внести не получилось

- Посмотреть логи SELinux утилитой audit2why
```
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
```

>- На клиенте отсутствуют ошибки

- Не закрывая сессию на клиенте, подключиться к серверу ns01 и проверить логи SELinux
```
elena_leb@ubuntunbleb:~/CLONE_SELinux/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Thu Jan 11 06:45:26 2024 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
```
- Утилита sealert (анализ ошибок в логе) выдает
```
[vagrant@ns01 ~]$ sealert -a /var/log/audit/audit.log
[Errno 13] Permission denied: '/var/log/audit/audit.log'

```
```
[root@ns01 ~]# cat /var/log/audit/audit.log | grep denied
type=AVC msg=audit(1705826797.574:1935): avc:  denied  { create } for  pid=5192 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
```

- Утилитой audit2why
>- audit2allow  -  создаёт  правила  политики SELinux allow/dontaudit из журналов отклонённых операций
>- audit2why - преобразовывает сообщения аудита SELinux в описание причины отказа  в  доступе (audit2allow -w)

```
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1705826797.574:1935): avc:  denied  { create } for  pid=5192 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# 

```
- **В логах указано , что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t**

- Проверим данную проблему в каталоге /etc/named
```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
- Посмотрим  в /var/log/messages
```
[root@ns01 ~]# tail -n 20  /var/log/messages | grep audit2allow
Jan 21 08:46:41 localhost python: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have create access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed create access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```
- В логе видно подсказку **ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012**

- **Selinux блокировал доступ к обновлению файлов для сервиса DNS и файлам к которым обращается DNS. Контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Возможные решения - скомпилировать модуль SELinux или измененить контекст безопасности.**

- Попробуем сформровать модуль
```
[root@ns01 ~]# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-iscworker0000.pp
```
- Посмотрим на него
```
[root@ns01 ~]# cat my-iscworker0000.te

module my-iscworker0000 1.0;

require {
	type etc_t;
	type named_t;
	class file create;
}

#============= named_t ==============

#!!!! WARNING: 'etc_t' is a base type.
allow named_t etc_t:file create;
```
- Видим предупреждение о изменение в базовом типе.

>- Базовый тип selinux - это универсальный тип, который, когда предоставляется, разрешает доступ к очень значительной части базовой системы. 
>- Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: sudo semanage fcontext -l | grep named

- Посмотрим в каком каталоге должны лежать файлы, чтобы на них распространялись правильные политики SELinux:
```bash
sudo semanage fcontext -l | grep named

/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0 
```
- Кроме место расположения, здесь видим тип контекста безопасности для каталогов - named_zone_t

- Изменим контекст безопасности для каталога /etc/named на named_zone_t
```
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

- Попробуем снова внести изменения с клиента:
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55089
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A
;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Jan 11 07:30:02 UTC 2024
;; MSG SIZE  rcvd: 96
```

- Изменения применились. Для проверки перезагружаем хосты и ещё раз сделать запрос с помощью dig
```
[vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7344
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10
;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Jan 11 07:38:55 UTC 2024
;; MSG SIZE  rcvd: 96
```

- **После перезагрузки настройки сохранились.**

>- Для того, чтобы вернуть правила обратно, можно ввести команду: restorecon -v -R /etc/named

- **Для того, чтобы SELinux корректно работал даже после изменения меток файловых систем, необходимо присвоить контекст named_zone_t всем файлам, находящимся в катологе /etc/named. Команда: semanage fcontext -a -t named_zone_t "/etc/named(/.*)?"**

```
[root@ns01 ~]# semanage fcontext -a -t named_zone_t "/etc/named(/.*)?"
```
- Проверяем.

```bash
[root@ns01 ~]# restorecon -v -R /etc/named 
[root@ns01 ~]# systemctl reboot
Connection to 127.0.0.1 closed by remote host.
elena_leb@ubuntunbleb:~/CLONE_SELinux/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Thu Jan 11 13:38:49 2024 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
- **На сервере контент сохранился**

- **Новая запись с клиента добавлена**
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10                                                
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.16
> send
> quit

```
- **Для решения выбран метод измененения контекста безопасности на каталоге, где лежат конфигурационные файлы.**

