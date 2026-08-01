#  Systemd — создание unit-файла

# Домашнее задание
1) Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).
2) Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
3) Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.

1) Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова

  Cоздаём файл с конфигурацией для сервиса в директории /etc/default - из неё сервис будет брать необходимые переменные.
    
    root@ubuntu:~# nano /etc/default/watchlog
    # Configuration file for my watchlog service
    # Place it to /etc/default
    # File and word in that file that we will be monit
    WORD="ALERT"
    LOG=/var/log/watchlog.log

  Cоздадим /var/log/watchlog.log и пишем туда строки на своё усмотрение,
плюс ключевое слово ‘ALERT’

    root@ubuntu:~# nano /opt/watchlog.sh
    #!/bin/bash

    WORD=$1
    LOG=$2
    DATE=`date`

    if grep $WORD $LOG &> /dev/null
    then
    logger "$DATE: I found word, Master!"
    else
    exit 0
    fi

  Права на запуск:
  
    root@ubuntu:~# chmod +x /opt/watchlog.sh 

  Создадим юнит для сервиса:

    root@ubuntu:~# nano /etc/systemd/system/watchlog.service
    [Unit]
    Description=My watchlog service

    [Service]
    Type=oneshot
    EnvironmentFile=/etc/default/watchlog
    ExecStart=/opt/watchlog.sh $WORD $LOG

  Создадим юнит для таймера:
    
    root@ubuntu:~# nano /etc/systemd/system/watchlog.timer
    [Unit]
    Description=Run watchlog script every 30 second

    [Timer]
    # Run every 30 second
    OnUnitActiveSec=30
    Unit=watchlog.service

    [Install]
    WantedBy=multi-user.target

  Запускаем timer:
  
    systemctl start watchlog.timer

  Проверка работы 

    root@ubuntu:~# tail -n 1000 /var/log/syslog  | grep word
    
    Aug  1 06:29:25 ubuntu kernel: [    7.505534] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
    Aug  1 06:33:52 ubuntu user: Sat Aug  1 06:33:52 UTC 2026: I found word, Master!
    Aug  1 06:35:07 ubuntu user: Sat Aug  1 06:35:07 UTC 2026: I found word, Master!
    Aug  1 06:36:15 ubuntu user: Sat Aug  1 06:36:15 UTC 2026: I found word, Master!
    Aug  1 06:37:54 ubuntu root: Sat 01 Aug 2026 06:37:54 AM UTC: I found word, Master!
    Aug  1 06:38:08 ubuntu root: Sat 01 Aug 2026 06:38:08 AM UTC: I found word, Master!
    Aug  1 06:39:04 ubuntu root: Sat 01 Aug 2026 06:39:04 AM UTC: I found word, Master!
    Aug  1 06:39:36 ubuntu root: Sat 01 Aug 2026 06:39:36 AM UTC: I found word, Master!
    Aug  1 06:40:06 ubuntu root: Sat 01 Aug 2026 06:40:06 AM UTC: I found word, Master!
    Aug  1 06:40:50 ubuntu root: Sat 01 Aug 2026 06:40:50 AM UTC: I found word, Master!
    Aug  1 06:41:21 ubuntu root: Sat 01 Aug 2026 06:41:21 AM UTC: I found word, Master!
    Aug  1 06:41:54 ubuntu root: Sat 01 Aug 2026 06:41:54 AM UTC: I found word, Master!
    Aug  1 06:42:30 ubuntu root: Sat 01 Aug 2026 06:42:30 AM UTC: I found word, Master!
    Aug  1 06:43:04 ubuntu root: Sat 01 Aug 2026 06:43:04 AM UTC: I found word, Master!
    Aug  1 06:43:35 ubuntu root: Sat 01 Aug 2026 06:43:35 AM UTC: I found word, Master!
    Aug  1 06:44:24 ubuntu root: Sat 01 Aug 2026 06:44:24 AM UTC: I found word, Master!

  2) Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта
  Устанавливаем spawn-fcgi и необходимые для него пакеты:

    root@ubuntu:~# apt install spawn-fcgi php php-cgi php-cli apache2  libapache2-mod-fcgid -y
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done

  Необходимо создать файл с настройками для будущего сервиса в файле /etc/spawn-fcgi/fcgi.conf

    root@ubuntu:~# mkdir /etc/spawn-fcgi
    
    root@ubuntu:~# nano /etc/spawn-fcgi/fcgi.conf
    # You must set some working options before the "spawn-fcgi" service will work.
    # If SOCKET points to a file, then this file is cleaned up by the init script.
    #
    # See spawn-fcgi(1) for all possible options.
    #
    # Example :
    SOCKET=/var/run/php-fcgi.sock
    OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"

  Юнит-файл для сервиса spawn-fcgi.service:

     root@ubuntu:~# nano /etc/systemd/system/spawn-fcgi.service
     [Unit]
    Description=Spawn-fcgi startup service by Otus
    After=network.target

    [Service]
    Type=simple
    PIDFile=/var/run/spawn-fcgi.pid
    EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
    ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
    KillMode=process

    [Install]
    WantedBy=multi-user.target


    root@ubuntu:~# systemctl daemon-reload 

    root@ubuntu:~# systemctl start spawn-fcgi.service 

    root@ubuntu:~# systemctl status spawn-fcgi.service 
    ● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: enabled)
     Active: active (running) since Sat 2026-08-01 07:02:06 UTC; 4s ago
    Main PID: 12618 (php-cgi)
      Tasks: 33 (limit: 4588)
     Memory: 14.4M
     CGroup: /system.slice/spawn-fcgi.service
             ├─12618 /usr/bin/php-cgi
             ├─12633 /usr/bin/php-cgi
             ├─12634 /usr/bin/php-cgi
             ├─12635 /usr/bin/php-cgi
             ├─12636 /usr/bin/php-cgi
             ├─12637 /usr/bin/php-cgi
             ├─12638 /usr/bin/php-cgi
             ├─12639 /usr/bin/php-cgi
             ├─12640 /usr/bin/php-cgi
             ├─12641 /usr/bin/php-cgi
             ├─12642 /usr/bin/php-cgi
             ├─12643 /usr/bin/php-cgi
             ├─12644 /usr/bin/php-cgi
             ├─12645 /usr/bin/php-cgi
             ├─12646 /usr/bin/php-cgi
             ├─12647 /usr/bin/php-cgi
             ├─12648 /usr/bin/php-cgi
             ├─12649 /usr/bin/php-cgi
             ├─12650 /usr/bin/php-cgi
             ├─12651 /usr/bin/php-cgi
             ├─12652 /usr/bin/php-cgi
             ├─12653 /usr/bin/php-cgi
             ├─12654 /usr/bin/php-cgi
             ├─12655 /usr/bin/php-cgi
             ├─12656 /usr/bin/php-cgi
             ├─12657 /usr/bin/php-cgi
             ├─12658 /usr/bin/php-cgi
             ├─12659 /usr/bin/php-cgi
             ├─12660 /usr/bin/php-cgi
             ├─12661 /usr/bin/php-cgi
             ├─12662 /usr/bin/php-cgi
             ├─12663 /usr/bin/php-cgi
             └─12664 /usr/bin/php-cgi
 
    Aug 01 07:02:06 ubuntu systemd[1]: Started Spawn-fcgi startup service by Otus.

  3) Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.
    Установим Nginx из стандартного репозитория:

    root@ubuntu:~# apt install nginx -y
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    
  Для запуска нескольких экземпляров сервиса модифицируем исходный service для использования различной конфигурации, а также PID-файлов. Для этого создадим новый Unit для работы с шаблонами (/etc/systemd/system/nginx@.service):

    root@ubuntu:~# nano /etc/systemd/system/nginx@.service
    # Stop dance for nginx
    # =======================
    #
    # ExecStop sends SIGSTOP (graceful stop) to the nginx process.
    # If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
    # and sends SIGTERM (fast shutdown) to the main process.
    # After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
    # SIGKILL to all the remaining processes in the process group (KillMode=mixed).
    #
    # nginx signals reference doc:
    # http://nginx.org/en/docs/control.html
    #
    [Unit]
    Description=A high performance web server and a reverse proxy server
    Documentation=man:nginx(8)
    After=network.target nss-lookup.target
    
    [Service]
    Type=forking
    PIDFile=/run/nginx-%I.pid
    ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
    ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
    ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
    ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
    TimeoutStopSec=5
    KillMode=mixed
    
    [Install]
    WantedBy=multi-user.target

  Далее необходимо создать два файла конфигурации (/etc/nginx/nginx-first.conf, /etc/nginx/nginx-second.conf). Их можно сформировать из стандартного конфига /etc/nginx/nginx.conf, с модификацией путей до PID-файлов и разделением по портам:
  
    root@ubuntu:~# nano /etc/nginx/nginx-first.conf
    pid /run/nginx-first.pid;

    http {
    …
    	server {
    		listen 9001;
    	}
    #include /etc/nginx/sites-enabled/*;
    ….
    }
#
    root@ubuntu:~# nano /etc/nginx/nginx-second.conf
    pid /run/nginx-first.pid;

    http {
    …
    	server {
    		listen 9005;
    	}
    #include /etc/nginx/sites-enabled/*;
    ….
    }    
#
  Запуск сервисов: 
  
    root@ubuntu:~# systemctl start nginx@first
    root@ubuntu:~# systemctl start nginx@second

  Проверка статуса: 

    ● nginx@second.service - A high performance web server and a reverse proxy server
         Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; vendor preset: enabled)
         Active: active (running) since Sat 2026-08-01 07:27:35 UTC; 7s ago
           Docs: man:nginx(8)
        Process: 13957 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-second.conf -q -g daemon on; master_process on; (code=>
        Process: 13974 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on; (code=exited, s>
       Main PID: 13975 (nginx)
          Tasks: 3 (limit: 4588)
         Memory: 3.4M
         CGroup: /system.slice/system-nginx.slice/nginx@second.service
                 ├─13975 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
                 ├─13976 nginx: worker process
                 └─13977 nginx: worker process
    
    Aug 01 07:27:35 ubuntu systemd[1]: Starting A high performance web server and a reverse proxy server...
    Aug 01 07:27:35 ubuntu systemd[1]: Started A high performance web server and a reverse proxy server.
    root@ubuntu:~# systemctl status nginx@first
    ● nginx@first.service - A high performance web server and a reverse proxy server
         Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; vendor preset: enabled)
         Active: active (running) since Sat 2026-08-01 07:27:27 UTC; 21s ago
           Docs: man:nginx(8)
        Process: 13930 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-first.conf -q -g daemon on; master_process on; (code=e>
        Process: 13951 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on; (code=exited, st>
       Main PID: 13952 (nginx)
          Tasks: 3 (limit: 4588)
         Memory: 5.2M
         CGroup: /system.slice/system-nginx.slice/nginx@first.service
                 ├─13952 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
                 ├─13953 nginx: worker process
                 └─13954 nginx: worker process

  Проверка портов:

      root@ubuntu:~# ss -tnulp | grep nginx
      tcp    LISTEN  0       511                   0.0.0.0:9001         0.0.0.0:*      users:(("nginx",pid=13954,fd=6),("nginx",pid=13953,fd=6),("nginx",pid=13952,fd=6))
      tcp    LISTEN  0       511                   0.0.0.0:9005         0.0.0.0:*      users:(("nginx",pid=13977,fd=6),("nginx",pid=13976,fd=6),("nginx",pid=13975,fd=6))
     
  Проверка процессов nginx:

      root@ubuntu:~# ps afx | grep nginx
      14108 pts/0    S+     0:00                      \_ grep --color=auto nginx
      13952 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
      13953 ?        S      0:00  \_ nginx: worker process
      13954 ?        S      0:00  \_ nginx: worker process
      13975 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
      13976 ?        S      0:00  \_ nginx: worker process
      13977 ?        S      0:00  \_ nginx: worker process
    
   Видим две группы процессов. 

   Задание выполнено.

    artem@mechrevo:~$ curl -I http://192.168.122.2:9001
    HTTP/1.1 200 OK
    Server: nginx/1.18.0 (Ubuntu)
    Date: Sat, 01 Aug 2026 08:13:18 GMT
    Content-Type: text/html
    Content-Length: 790
    Last-Modified: Sat, 01 Aug 2026 08:11:36 GMT
    Connection: keep-alive
    ETag: "6a6daa38-316"
    Accept-Ranges: bytes
---
    artem@mechrevo:~$ curl -I http://192.168.122.2:9005
    HTTP/1.1 200 OK
    Server: nginx/1.18.0 (Ubuntu)
    Date: Sat, 01 Aug 2026 08:13:27 GMT
    Content-Type: text/html
    Content-Length: 790
    Last-Modified: Sat, 01 Aug 2026 08:11:36 GMT
    Connection: keep-alive
    ETag: "6a6daa38-316"
    Accept-Ranges: bytes
---  
    artem@mechrevo:~$ curl -I http://192.168.122.2:80
    curl: (7) Failed to connect to 192.168.122.2 port 80 after 0 ms: Could not connect to server
---
