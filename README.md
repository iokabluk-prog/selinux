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

