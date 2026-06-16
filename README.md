# 1.  Запуск nginx на нестандартном порту 3-мя разными способами 
# После развертывания ВМ nginx не запускается
[root@192 ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
[root@192 ~]# systemctl status nginx
× nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: failed (Result: exit-code) since Tue 2026-06-16 12:20:36 UTC; 6min ago
    Process: 19474 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 19475 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
        CPU: 64ms

Jun 16 12:20:36 192.168.1.8 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 16 12:20:36 192.168.1.8 nginx[19475]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 16 12:20:36 192.168.1.8 nginx[19475]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission den>
Jun 16 12:20:36 192.168.1.8 nginx[19475]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jun 16 12:20:36 192.168.1.8 systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILU>
Jun 16 12:20:36 192.168.1.8 systemd[1]: nginx.service: Failed with result 'exit-code'.
Jun 16 12:20:36 192.168.1.8 systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
# Проверяем статуч firewalld
[root@192 ~]# systemctl status firewalld
○ firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: inactive (dead)
       Docs: man:firewalld(1)
# Проверяем конфигурацию nginx
[root@192 ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
# Проверяем режим работы SELinux
[root@192 ~]# getenforce
Enforcing

# Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool
# Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта
[root@192 ~]# cat /var/log/audit/audit.log |grep 4881
type=AVC msg=audit(1781612436.520:189): avc:  denied  { name_bind } for  pid=19475 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
# Копируем время 1781612436.520:189
# С помощью grep 1781612436.520:189 /var/log/audit/audit.log | audit2why проверяем
[root@192 ~]# grep 1781612436.520:189 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1781612436.520:189): avc:  denied  { name_bind } for  pid=19475 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
# Видим, что необходимо изменить параметр  nis_enabled
[root@192 ~]# setsebool -P nis_enabled on
[root@192 ~]# systemctl restart nginx
[root@192 ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Tue 2026-06-16 12:42:39 UTC; 10s ago
    Process: 19923 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 19924 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 19925 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 19926 (nginx)
      Tasks: 3 (limit: 12012)
     Memory: 2.9M
        CPU: 105ms
     CGroup: /system.slice/nginx.service
             ├─19926 "nginx: master process /usr/sbin/nginx"
             ├─19927 "nginx: worker process"
             └─19928 "nginx: worker process"

Jun 16 12:42:39 192.168.1.8 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 16 12:42:39 192.168.1.8 nginx[19924]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 16 12:42:39 192.168.1.8 nginx[19924]: nginx: configuration file /etc/nginx/nginx.conf test is successf>
Jun 16 12:42:39 192.168.1.8 systemd[1]: Started The nginx HTTP and reverse proxy server.
# Скрин, что nginx работает по http://192.168.1.8:4881/
<img width="1736" height="760" alt="image" src="https://github.com/user-attachments/assets/951eeba5-9669-4a79-9b47-f7167c93165c" />
[root@192 ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
# Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled: setsebool -P nis_enabled off
[root@192 ~]# setsebool -P nis_enabled off
[root@192 ~]# getsebool -a | grep nis_enabled
nis_enabled --> off
[root@192 ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.

# Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип
[root@192 ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
# Добавим порт в тип http_port_t: semanage port -a -t http_port_t -p tcp 4881
[root@192 ~]# semanage port -a -t http_port_t -p tcp 4881
[root@192 ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
# Теперь перезапускаем службу nginx и проверим её работу
[root@192 ~]# systemctl restart nginx
[root@192 ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Tue 2026-06-16 12:55:12 UTC; 10s ago
    Process: 19977 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 19978 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 19979 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 19980 (nginx)
      Tasks: 3 (limit: 12012)
     Memory: 2.9M
        CPU: 107ms
     CGroup: /system.slice/nginx.service
             ├─19980 "nginx: master process /usr/sbin/nginx"
             ├─19981 "nginx: worker process"
             └─19982 "nginx: worker process"

Jun 16 12:55:12 192.168.1.8 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 16 12:55:12 192.168.1.8 nginx[19978]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 16 12:55:12 192.168.1.8 nginx[19978]: nginx: configuration file /etc/nginx/nginx.conf test is successf>
Jun 16 12:55:12 192.168.1.8 systemd[1]: Started The nginx HTTP and reverse proxy server.
# Скрин, что nginx работает по http://192.168.1.8:4881/
<img width="1713" height="673" alt="image" src="https://github.com/user-attachments/assets/fc865bf5-012d-4b6c-abec-b3a3e99f0c45" />
# Удаляем нестандартный порт из имеющегося типа можно с помощью команды: semanage port -d -t http_port_t -p tcp 4881
[root@192 ~]# semanage port -d -t http_port_t -p tcp 4881
[root@192 ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@192 ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
[root@192 ~]# systemctl status nginx
× nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: failed (Result: exit-code) since Tue 2026-06-16 12:58:11 UTC; 27s ago
   Duration: 2min 58.558s
    Process: 19999 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 20001 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
        CPU: 40ms

Jun 16 12:58:11 192.168.1.8 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 16 12:58:11 192.168.1.8 nginx[20001]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 16 12:58:11 192.168.1.8 nginx[20001]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission den>
Jun 16 12:58:11 192.168.1.8 nginx[20001]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jun 16 12:58:11 192.168.1.8 systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILU>
Jun 16 12:58:11 192.168.1.8 systemd[1]: nginx.service: Failed with result 'exit-code'.
Jun 16 12:58:11 192.168.1.8 systemd[1]: Failed to start The nginx HTTP and reverse proxy server.

# Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux
# Посмотрим логи SELinux, которые относятся к Nginx
[root@192 ~]# grep nginx /var/log/audit/audit.log
type=SERVICE_START msg=audit(1781614691.100:263): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'UID="root" AUID="unset"
# Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту
[root@192 ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
# Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль: semodule -i nginx.pp
[root@192 ~]# semodule -i nginx.pp
[root@192 ~]# systemctl start nginx
[root@192 ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Tue 2026-06-16 13:03:20 UTC; 17s ago
    Process: 20053 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 20054 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 20055 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 20056 (nginx)
      Tasks: 3 (limit: 12012)
     Memory: 2.9M
        CPU: 98ms
     CGroup: /system.slice/nginx.service
             ├─20056 "nginx: master process /usr/sbin/nginx"
             ├─20057 "nginx: worker process"
             └─20058 "nginx: worker process"

Jun 16 13:03:20 192.168.1.8 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 16 13:03:20 192.168.1.8 nginx[20054]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 16 13:03:20 192.168.1.8 nginx[20054]: nginx: configuration file /etc/nginx/nginx.conf test is successf>
Jun 16 13:03:20 192.168.1.8 systemd[1]: Started The nginx HTTP and reverse proxy server.
# После добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки. 
# Просмотр всех установленных модулей: semodule -l
[root@192 ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
[root@192 ~]#

 







