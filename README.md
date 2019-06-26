# roger-skyline-1
Basics commands and first reflexes in system and network administration.


# Roger-skyline-1

##### 1.	Установка виртуальной машины и ОС

- скачать образ отсюда [Debian] (debian-9.9.0-i386-netinst.iso)
- открыть VirtualBox и создать VM в goinfre в папке юзера (твоей папке) на этапе “File location and size”
Размер диска - 8 ГБ@
- в ходе установки разбить диск согласно условиям задания
- Я создал три диска:
-- 4,5 Гб Первичный 
-- 1,2 ГБ Логический 
-- 2,9 ГБ Логический

Такое разбиение поможет выполнить условие задания и Первичный диск “/” будет весить 4,2 ГБ
В разделе “Software selection” выбираю только SSH server и Standart system utilities
С этой частью все. Посмотреть итог lsblk

##### 2.  	Проверка обновлений и настройка sudo

В терминали ВМ пишем следющее: 
```
$ su
$ apt update
$ apt upgrade
```
- устанавливаем то, чем будем пользоваться в будущем:
```
$ apt-get install sudo vim ufw portsentry fail2ban apache2 net-tools mailutils -y
```
- редактируешь файл vim /etc/sudoers
```
# User privilege specification
root    	ALL=(ALL:ALL) ALL
prawney     ALL=(ALL:ALL) ALL
```
- на выходе жмешь `ESC` и набираешь `:wq!`
```
$ adduser prawney sudo
```
##### 3. 	Установка статического IP
- отредактируй файл `sudo vim /etc/network/interfaces` следующим образом:
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).
source /etc/network/interfaces.d/*
# The loopback network interface
auto lo
iface lo inet loopback
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet static
	address 10.0.2.15
	netmask 30
	gateway 10.0.2.2
```

- теперь перезапускаем командой:
```
$ sudo /etc/init.d/networking restart
```
- поднимаем командой:
```
ifup enp0s3
```
##### 4. 	Настройка SSH

- Редактируем файл на ВМ vim /etc/ssh/sshd_config следующим образом
```
port 90
PasswordAuthentification yes
```
- теперь перезапускаем командой:
```
$ sudo /etc/init.d/ssh restart
```
- Подключишся с мака 
```
$ ssh prawney@127.0.0.1 -p 9090
```
- В терминале на маке генирируешь ключи
```
ssh-keygen -t rsa
```
- Отправляешь их на машину
```
ssh-copy-id -i id_rsa.pub prawney@127.0.0.1 -p 9090
```
- Опять редактируем файл на ВМ vim /etc/ssh/sshd_config следующим образом
```
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentification no
```
- теперь перезапускаем командой:
```
$ sudo /etc/init.d/ssh restart
```
##### 5. 	Firewall
- В терминале на ВМ пишешь:
```
$ sudo ufw status
$ sudo ufw enable
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 90/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 443
$ sudo ufw reload
```
Значения:
>SSH : sudo ufw allow 90/tcp 
>HTTP : sudo ufw allow 80/tcp
>HTTPS : sudo ufw allow 443
##### 6. 	DOS (Denial Of Service Attack) protection
- редактируешь файл следующим образом `sudo vim /etc/fail2ban/jail.conf`
```
[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
backend = %(sshd_backend)s
maxretry = 3
findtime = 21
bantime = 120

[sshd-ddos]
enabled = true
port    = ssh
logpath = /var/log/auth.log
backend = %(sshd_backend)s
maxretry = 3
findtime = 21
bantime = 120

[apache-auth]
enabled  = true
port     = http,https
logpath  = %(apache_error_log)s

[apache-badbots]
enabled  = true
port     = http,https
logpath  = %(apache_access_log)s
bantime  = 172800
maxretry = 1

[apache-noscript]
enabled  = true
port     = http,https
logpath  = %(apache_error_log)s

[apache-overflows]
enabled  = true
port     = http,https
logpath  = %(apache_error_log)s
maxretry = 2
```
Значения:
**logpath** — полный путь к файлу, в который будет записываться информация о попытках получения доступа.
**findtime** — время в секундах, в течение которого наблюдается подозрительная активность;
**maxretry** — разрешенное количество повторных попыток подключения к серверу;
**bantime** — промежуток времени, в течение которого попавший в черный список IP будет оставаться заблокированным.
- теперь перезапускаем командой:
```
$ sudo /etc/init.d/fail2ban restart
```
- **первое* смотрит состояние, **второе* для разблокировки sshd, **третье* для разблокировки apache
```
sudo fail2ban-client status
sudo fail2ban-client set sshd unbanip 10.0.2.2
sudo fail2ban-client set apache-dos unbanip 10.0.2.2
```

##### 7. 	Protection port scans
- Редактируешь файл следующим образом `sudo vim /etc/default/portsentry`
```
TCP_MODE="atcp"
UDP_MODE="audp"
```
- Редактируешь файл следующим образом `sudo vim /etc/portsentry/portsentry.conf`
```
BLOCK_UDP="1"
BLOCK_TCP="1"
# iptables support for Linux
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
#KILL_HOSTS_DENY="ALL: $TARGET$ : DENY"
```
- Перезагружаем командой:
```
sudo /etc/init.d/portsentry restart
```

 ##### 8.	Stop the services
 - Смотрим наши сервисы:
 ```
ls -l /etc/init.d
```
- Активные сервисы:
```
sudo service --status-all
```
- Более подробно:
```
sudo systemctl --full --type service
```
- Отключаем, так же отключаем автозагрузку:
```
sudo service service_name stop
sudo systemctl disable service_name
```

##### 9.	 Update Packages
- Создаем файл для логов:  `sudo vim /var/log/update_script.log`
- Создаем скрипт: `sudo vim /root/update_script.sh`:
```
#! / bin / bash 
apt update -y >> /var/log/update_script.log
apt upgrade -y >> /var/log/update_script.log
```
- Меняем права `sudo chmod 755 /root/update_script.sh`
- Автоматизируем через `sudo crontab -e`:
```
0 4 * * 1 root /root/update_script.sh
@reboot root /root/update_script.sh
```
##### 10. 	Monitor Crontab Changes
- Создаем скрипт `sudo vim /root/crontab_monitor.sh`:
```
#!/bin/bash

DIFF=$(diff /etc/crontab_save /etc/crontab)

cat /etc/crontab > /etc/crontab_save

if [ "$DIFF" != ""] then
    echo "crontab check: changed, notifying admin."
    sendmail root@127.0.0.1 < ~/mail.txt
else
    echo "crontab check: unchanged."
fi
```
- Меняем права `sudo chmod 755 /root/crontab_monitor.sh`
- Автоматизируем через `sudo crontab -e`:
```
0 0 * * * root /root/scripts/crontab_monitor.sh
```
- Создаем свой mail.txt, например `Fail /etc/crontab changed.`
- Включаешь крон `sudo systemctl enable cron`
##### 11.	Web Part
- Создаем сайт и перемещаем его в  `/var/www/html/index.html`
- Подключаемся и проверяем

##### 12.	Configure SSL
- Генерируем самоподписанный ключ `sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt`
- Cоздаем файл  `sudo vim /etc/apache2/ssl-params.conf`:
```
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3
SSLHonorCipherOrder On
Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains; preload"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
# Requires Apache >= 2.4
SSLCompression off
SSLSessionTickets Off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
SSLOpenSSLConfCmd DHParameters "/etc/ssl/certs/dhparam.pem"
```
- В файле `sudo vim /etc/apache2/sites-available/default-ssl.conf` редактируем:
```
ServerAdmin prawney@21-school.ru
ServerName	127.0.0.1
SSLCertificateFile	/etc/ssl/certs/apache-selfsigned.crt
SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

BrowserMatch "MSIE [2-6]" \
		nokeepalive ssl-unclean-shutdown \
		downgrade-1.0 force-response-1.0
```
- В файл `sudo vim /etc/apache2/sites-available/000-default.conf` дописываем:
```
Redirect "/" "https://127.0.0.1/"
```
- Добавдяем в фаерволл https и перезапускаем:
 ```
 sudo ufw allow 4443
 sudo ufw reload
 ```
 - Настраиваем ssl в apache:
 ```
sudo apache2ctl configtest
sudo a2enmod ssl
sudo a2enmod headers
sudo a2ensite default-ssl
sudo a2enconf ssl-params
sudo systemctl reload apache2
```
##### 13. 	Part Deployment
- Копируем репозиторий на Mac и VM

- Пушим сайт на Git

- На VM в файле `sudo vim /etc/apache2/sites-available/default-ssl.conf` меняешь 
`var/www/html/`
на
`/home/prawney/code21/web_roger`
 аналогично предыдущему пункту меняешь в файле `sudo vim /etc/apache2/sites-available/000-default.conf` 

- В файле `sudo vim /etc/apache2/apache2.conf` меняешь следующие строки
```
<Directory />
	Options FollowSymLinks
	AllowOverride None
	Require all granted
</Directory>

<Directory /usr/share>
	AllowOverride None
	Require all granted
</Directory>

<Directory /home/prawney/code21/web_roger>
	Options Indexes FollowSymLinks
	AllowOverride None
	Require all granted
</Directory>
```
`*/home/prawney/code21/web_roger` - это папка с сайтом склоненная с Git'a

- Теперь любые изменения которые мы запушили с Мака, можно сразу получить на VM и увидеть на сайте командой git pull.
***
**Спасибо за внимание!**

   [Debian]: <https://cdimage.debian.org/debian-cd/current/i386/iso-cd/>

