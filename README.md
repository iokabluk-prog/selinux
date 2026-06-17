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

# 2. Обеспечение работоспособности приложения при включенном SELinux
# Выполним установку ansible и git
root@ubuntu01:~# sudo apt install ansible -y
root@ubuntu01:~# sudo apt install git
# Выполним клонирование репозитория
git clone https://github.com/Nickmob/vagrant_selinux_dns_problems.git
# Развернём 2 ВМ с помощью vagrant
# Vagrant в WSL  с VirtualBox, установленным в Windows
ubuntuadmin@WS01:~/vagrant_project$ vagrant up client
# Содержание Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
  config.vm.box = "almalinux/9"
  config.vm.box_url = "file:///mnt/c/vagrant_projects/my_vm/vagrant_selinux_dns_problems/package.box"

  config.vm.boot_timeout = 600

  config.vm.provider "virtualbox" do |v|
    v.memory = 2048
    v.cpus = 2
    v.gui = false
    v.customize ["modifyvm", :id, "--graphicscontroller", "vmsvga"]
    v.customize ["modifyvm", :id, "--accelerate3d", "off"]
    v.customize ["modifyvm", :id, "--vram", "16"]
  end

  config.vm.define "ns01" do |ns01|
    ns01.vm.synced_folder ".", "/vagrant", disabled: true
    ns01.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "dns"
    ns01.vm.hostname = "ns01"

    # Используем реальный IP
    ns01.ssh.host = "192.168.1.6"
    ns01.ssh.port = 22
    ns01.ssh.username = "vagrant"
    # НЕ УКАЗЫВАЕМ private_key_path - Vagrant сам создаст ключи

    ns01.vm.provision "shell", inline: <<-SHELL
      echo "blacklist vboxvideo" > /etc/modprobe.d/blacklist-vboxvideo.conf
      systemctl restart sshd
      dracut --force
    SHELL
  end

  config.vm.define "client" do |client|
    client.vm.synced_folder ".", "/vagrant", disabled: true
    client.vm.network "private_network", ip: "192.168.50.15", virtualbox__intnet: "dns"
    client.vm.hostname = "client"

    client.ssh.host = "192.168.1.7"  # или другой IP
    client.ssh.port = 22
    client.ssh.username = "vagrant"
    # НЕ УКАЗЫВАЕМ private_key_path

    client.vm.provision "shell", inline: <<-SHELL
      echo "blacklist vboxvideo" > /etc/modprobe.d/blacklist-vboxvideo.conf
      systemctl restart sshd
      dracut --force
    SHELL
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "provisioning/playbook.yml"
    ansible.become = "true"
    ansible.limit = "all"
    ansible.verbose = "v"
    ansible.groups = {
      "dns_servers" => ["ns01"],
      "clients" => ["client"]
    }
  end
 # Проверяем статус ВМ
 ubuntuadmin@WS01:~/vagrant_project$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
# Подключимся к клиенту
ubuntuadmin@WS01:~/vagrant_project$ vagrant ssh client
###############################
### Welcome to the DNS lab! ###
###############################

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

###############################
### Enjoy! ####################
###############################
Last login: Wed Jun 17 10:40:47 2026 from 192.168.1.5
[vagrant@client ~]$
# Попробуем внести изменения в зону
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
# Изменения внести не получилось. Давайте посмотрим логи SELinux, чтобы понять в чём может быть проблема
[root@client ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1781691313.814:609): avc:  denied  { dac_read_search } for  pid=3362 comm="20-chrony-dhcp" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.


# Процесс: 20-chrony-dhcp (PID 3424) — это скрипт NetworkManager, который запускается при изменении сети для настройки chrony (службы синхронизации времени)

# Действие запрещено: { dac_read_search } — процесс пытается обойти обычные права доступа к файлам (DAC = Discretionary Access Control)

# Возможность (capability): capability=2 (CAP_DAC_READ_SEARCH) — процессу нужно читать файлы и каталоги, к которым у него нет прямого доступа

# Контекст: И источник, и цель — NetworkManager_dispatcher_chronyc_t (один и тот же контекст)

# Статус: permissive=0 — SELinux включён в режиме enforcing и блокирует действие
# Сгенерировать модуль на основе сообщения из лога
echo "type=AVC msg=audit(1781691316.405:624): avc:  denied  { dac_read_search } for  pid=3424 comm=\"20-chrony-dhcp\" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0" | audit2allow -M chrony-dhcp-dac
# Установить созданный модуль
sudo semodule -i chrony-dhcp-dac.pp
# Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux
[vagrant@ns01 ~]$ cat /var/log/audit/audit.log | audit2why
cat: /var/log/audit/audit.log: Permission denied
Nothing to do
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1781686028.947:631): avc:  denied  { dac_read_search } for  pid=3414 comm="20-chrony-dhcp" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
# Ошибка та же
# Создать модуль из конкретного сообщения на клиенте
echo "type=AVC msg=audit(1781686028.947:631): avc:  denied  { dac_read_search } for  pid=3414 comm=\"20-chrony-dhcp\" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0" | audit2allow -M chrony-dhcp-dac

# Установить на  клиенте
sudo semodule -i chrony-dhcp-dac.pp
# Модуль добавляет правило на клиенте
allow NetworkManager_dispatcher_chronyc_t self:capability dac_read_search;
# Это разрешает процессу с этим контекстом использовать capability CAP_DAC_READ_SEARCH
# Выполним перезагрузку клиента
# Полсе перезагрузки осталась ошибка
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1781696176.968:34): avc:  denied  { open } for  pid=829 comm="20-chrony-dhcp" path="/etc/sysconfig/network-scripts/ifcfg-eth1" dev="sda4" ino=17015107 scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=unconfined_u:object_r:user_tmp_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
# Файл /etc/sysconfig/network-scripts/ifcfg-eth1 имеет контекст user_tmp_t (временный пользовательский), а должен иметь контекст net_conf_t или etc_t.
scontext: NetworkManager_dispatcher_chronyc_t (процесс)
tcontext: user_tmp_t (файл) ← НЕПРАВИЛЬНЫЙ КОНТЕКСТ!
# Посмотреть текущий контекст
[root@client ~]# ls -Z /etc/sysconfig/network-scripts/ifcfg-eth1
unconfined_u:object_r:user_tmp_t:s0 /etc/sysconfig/network-scripts/ifcfg-eth1
# Восстановить правильный контекст для всех файлов в каталоге
[root@client ~]# restorecon -Rv /etc/sysconfig/network-scripts/
Relabeled /etc/sysconfig/network-scripts/ifcfg-eth1 from unconfined_u:object_r:user_tmp_t:s0 to unconfined_u:object_r:net_conf_t:s0
[root@client ~]# ls -Z /etc/sysconfig/network-scripts/ifcfg-eth1
unconfined_u:object_r:net_conf_t:s0 /etc/sysconfig/network-scripts/ifcfg-eth1
# После перезагрузки на клиенте ошибок нет
# Подключаемся к ns01 и проверим логи SELinux
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1781686028.947:631): avc:  denied  { dac_read_search } for  pid=3414 comm="20-chrony-dhcp" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# echo "type=AVC msg=audit(1781686028.947:631): avc:  denied  { dac_read_search } for  pid=3414 comm=\"20-chrony-dhcp\" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0" | audit2allow -M chrony-dhcp-dac
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i chrony-dhcp-dac.pp

[root@ns01 ~]# sudo semodule -i chrony-dhcp-dac.pp
# Перезагрузим сервер
# Осталась ошибка 
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why | grep -E "denied|ERROR|Permission" | tail -n 1
type=AVC msg=audit(1781697418.288:79): avc:  denied  { write } for  pid=806 comm="isc-net-0001" name="dynamic" dev="sda4" ino=16864157 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0
# В логах мы видим, что ошибка в контексте безопасности. Целевой контекст named_conf_t
# Для сравнения посмотрим существующую зону (localhost) и её контекст
[root@ns01 ~]# ls -alZ /var/named/named.localhost
-rw-r-----. 1 root named system_u:object_r:named_zone_t:s0 152 Jun 10 13:05 /var/named/named.localhost
# У наших конфигов в /etc/named вместо типа named_zone_t используется тип named_conf_t.
# Проверим данную проблему в каталоге /etc/named:
[root@ns01 ~]# ls -laZ /etc/named
total 28
drw-rwx---.  3 root named system_u:object_r:named_conf_t:s0      121 Jun 17 08:56 .
drwxr-xr-x. 87 root root  system_u:object_r:etc_t:s0            8192 Jun 17 11:55 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_conf_t:s0   56 Jun 17 08:56 dynamic
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      805 Jun 17 08:56 named.50.168.192.rev
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      628 Jun 17 08:56 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      626 Jun 17 08:55 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      676 Jun 17 08:56 named.newdns.lab
# Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: sudo semanage fcontext -l | grep named
[root@ns01 ~]# sudo semanage fcontext -l | grep named
/dev/gpmdata                                       named pipe         system_u:object_r:gpmctl_t:s0
/dev/initctl                                       named pipe         system_u:object_r:initctl_t:s0
/dev/xconsole                                      named pipe         system_u:object_r:xconsole_device_t:s0
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0
/etc/named(/.*)?                                   all files          system_u:object_r:named_conf_t:s0
/etc/named\.caching-nameserver\.conf               regular file       system_u:object_r:named_conf_t:s0
/etc/named\.conf                                   regular file       system_u:object_r:named_conf_t:s0
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0
/etc/named\.root\.hints                            regular file       system_u:object_r:named_conf_t:s0
/etc/rc\.d/init\.d/named                           regular file       system_u:object_r:named_initrc_exec_t:s0
/etc/rc\.d/init\.d/named-sdb                       regular file       system_u:object_r:named_initrc_exec_t:s0
/etc/rc\.d/init\.d/unbound                         regular file       system_u:object_r:named_initrc_exec_t:s0
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0
/usr/lib/systemd/system/named-sdb.*                regular file       system_u:object_r:named_unit_file_t:s0
/usr/lib/systemd/system/named.*                    regular file       system_u:object_r:named_unit_file_t:s0
/usr/lib/systemd/system/unbound.*                  regular file       system_u:object_r:named_unit_file_t:s0
/usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0
/usr/sbin/lwresd                                   regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named                                    regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-checkconf                          regular file       system_u:object_r:named_checkconf_exec_t:s0
/usr/sbin/named-pkcs11                             regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-sdb                                regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/unbound                                  regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/unbound-anchor                           regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/unbound-checkconf                        regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/unbound-control                          regular file       system_u:object_r:named_exec_t:s0
/usr/share/munin/plugins/named                     regular file       system_u:object_r:services_munin_plugin_exec_t:s0
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0
/var/named/chroot/dev                              directory          system_u:object_r:device_t:s0
/var/named/chroot/dev/log                          socket             system_u:object_r:devlog_t:s0
/var/named/chroot/dev/null                         character device   system_u:object_r:null_device_t:s0
/var/named/chroot/dev/random                       character device   system_u:object_r:random_device_t:s0
/var/named/chroot/dev/urandom                      character device   system_u:object_r:urandom_device_t:s0
/var/named/chroot/dev/zero                         character device   system_u:object_r:zero_device_t:s0
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0
/var/named/chroot/etc/localtime                    regular file       system_u:object_r:locale_t:s0
/var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.conf                  regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.rfc1912.zones         regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/pki(/.*)?                    all files          system_u:object_r:cert_t:s0
/var/named/chroot/etc/rndc\.key                    regular file       system_u:object_r:dnssec_t:s0
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0
/var/named/chroot/proc(/.*)?                       all files          <<None>>
/var/named/chroot/run/named.*                      all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/usr/lib(/.*)?                    all files          system_u:object_r:lib_t:s0
/var/named/chroot/var/log                          directory          system_u:object_r:var_log_t:s0
/var/named/chroot/var/log/named.*                  regular file       system_u:object_r:named_log_t:s0
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0
/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/var/run/dbus(/.*)?               all files          system_u:object_r:system_dbusd_var_run_t:s0
/var/named/chroot/var/run/named.*                  all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/tmp(/.*)?                    all files          system_u:object_r:named_cache_t:s0
/var/named/chroot_sdb/dev                          directory          system_u:object_r:device_t:s0
/var/named/chroot_sdb/dev/null                     character device   system_u:object_r:null_device_t:s0
/var/named/chroot_sdb/dev/random                   character device   system_u:object_r:random_device_t:s0
/var/named/chroot_sdb/dev/urandom                  character device   system_u:object_r:urandom_device_t:s0
/var/named/chroot_sdb/dev/zero                     character device   system_u:object_r:zero_device_t:s0
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0
/var/named/named\.ca                               regular file       system_u:object_r:named_conf_t:s0
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0
/var/run/ecblp0                                    named pipe         system_u:object_r:cupsd_var_run_t:s0
/var/run/initctl                                   named pipe         system_u:object_r:initctl_t:s0
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0
/var/run/ndc                                       socket             system_u:object_r:named_var_run_t:s0
/var/run/systemd/initctl/fifo                      named pipe         system_u:object_r:initctl_t:s0
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/usr/lib64 = /usr/lib
/var/named/chroot/lib64 = /usr/lib
/var/named/chroot/var = /var

# Изменим тип контекста безопасности для каталога /etc/named: sudo chcon -R -t named_zone_t /etc/named

[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
total 28
drw-rwx---.  3 root named system_u:object_r:named_zone_t:s0      121 Jun 17 08:56 .
drwxr-xr-x. 87 root root  system_u:object_r:etc_t:s0            8192 Jun 17 11:55 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_zone_t:s0   56 Jun 17 08:56 dynamic
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      805 Jun 17 08:56 named.50.168.192.rev
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      628 Jun 17 08:56 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      626 Jun 17 08:55 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      676 Jun 17 08:56 named.newdns.lab

# Попробуем снова внести изменения с клиента:
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.16.23-RH <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 38632
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; AUTHORITY SECTION:
.                       790     IN      SOA     a.root-servers.net. nstld.verisign-grs.com. 2026061700 1800 900 604800 86400
# Перезагружаем клиента и сервер и проверяем настройки
[vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.16.23-RH <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37590
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 0fcd2c22daebb9b2010000006a328f63cd2970aade66d7e9 (good)
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; Query time: 23 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Wed Jun 17 12:13:23 UTC 2026
;; MSG SIZE  rcvd: 85

;; Query time: 17 msec
;; SERVER: 192.168.1.1#53(192.168.1.1)
;; WHEN: Wed Jun 17 12:09:16 UTC 2026
;; MSG SIZE  rcvd: 116
# Успешно выполнили DNS-запрос к серверу 192.168.50.10 для домена www.ddns.lab
