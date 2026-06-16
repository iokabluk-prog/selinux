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







