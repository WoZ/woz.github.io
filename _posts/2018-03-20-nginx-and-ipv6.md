---
layout: post
title: Nginx и ошибки resolve с IPv6.
tags: [linux, network, tuning, IPv6, IPv4, resolve, glibc, NSS, getaddrinfo]
---

Данная история с одной стороны поучительная и показывает почему внимательность к мелочам важна, а с другой стороны
обучающая и показывает как подходить к решению задач. Началось все с того, что при донастройке nginx в dev окружении,
когда я проводил эксперименты, были замечены ответы от сервера с кодами 5xx. То HLS чанк не отдался с первого раза,
то плейлист не загрузился. Вроде как на продакшне проблемы не наблюдаются... Но я решил закопаться глубже. Забегая
вперед отмечу, что эта придирка позволила обнаружить некорректное поведение на всех серверах где был выключен IPv6,
но часть сервисов использующих `glibc` для резолвинга адресов по прежнему пыталась достучаться по IPv6. 

Немного про архитектуру: на том же сервере где стоял nginx приствует сервис и nginx репроксирует запросы на него.
Сервис в одном экземпляре и в списке `upstream`'ов он один. nginx кэширует ответы и с помощью модуля
[auth](http://nginx.org/ru/docs/http/ngx_http_auth_request_module.html) проверяет разрешения на просмотр. Именно из-за
активного кэширования на продакшне было сложно словить эту ошибку.

Упрощенная конфигурация nginx приведена ниже. Я убрал HLS-specific вещи и оставил минимальный набор.

```
worker_processes  1;

error_log  /var/log/nginx/error.log  debug;

events {
    worker_connections  1024;
}

http {

    upstream nginx-controller {
        server localhost:8888 max_fails=0;
    }

    server {
        listen 80;

        location / {
            proxy_method 'GET';
            proxy_intercept_errors on;
            proxy_next_upstream 'off';
            proxy_connect_timeout '1s';
            proxy_pass http://nginx-controller;
        }
    }
}
```

Демо сервисом будет обычный netcat.

```
$ while true ; do nc -l -4 -p 8888 -c 'echo -e "HTTP/1.1 200 OK\n\n"'; done
```

Netcat будет слушать порт 8888, используя только ipv4 интерфейс. Убедиться в этом можно с помощью curl:

```
$ curl -i -g -6 "http://[::1]:8888/"
curl: (7) Failed connect to ::1:8888; Connection refused
$ curl -i "http://localhost:8888"
HTTP/1.1 200 OK


```

Теперь же попробуем обратиться через nginx.

```
[root@spotter_edge /]# curl -I -s "http://localhost" | head -1
HTTP/1.1 200 OK
[root@spotter_edge /]# curl -I -s "http://localhost" | head -1
HTTP/1.1 502 Bad Gateway
[root@spotter_edge /]# curl -I -s "http://localhost" | head -1
HTTP/1.1 200 OK
[root@spotter_edge /]# curl -I -s "http://localhost" | head -1
HTTP/1.1 502 Bad Gateway
```

Демо пример специально подготовлен для явной демонстрации поведения что я наблюдал. С реальным конфигом заметить было
сложнее и на протяжении месяца в продакшне не было замечено подобных отказов. Они были, но в меньшем обьеме и
трактовались как проблемы с доставляемости контента до конкретного клиента.

Первым делом я обратился к логам nginx и там обнаружил такие записи

```
2018/03/20 10:11:49 [error] 4130#4130: *11 connect() failed (111: Connection refused) while connecting to upstream,
client: 127.0.0.1, server: , request: "GET / HTTP/1.1", upstream: "http://[::1]:8888/"        , host: "localhost"
```

Запись `upstream: "http://[::1]:8888/"` намекает, что использовался IPv6, который `netcat` не слушал, как и продакшн
сервис. Но на хосте выключен IPv6!

В этом можно убедиться посмотрев параметры ядра и вывод `ifconfig`.

```
$ sysctl net.ipv6.conf.all.disable_ipv6 net.ipv6.conf.default.disable_ipv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

$ ifconfig -a
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.101.5  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 02:42:c0:a8:65:05  txqueuelen 0  (Ethernet)
        RX packets 15789549  bytes 2553533549 (2.3 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9783999  bytes 1318627420 (1.2 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1  (Local Loopback)
        RX packets 606047  bytes 67810892 (64.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 606047  bytes 67810892 (64.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Будь бы IPv6 включен, ifconfig показал бы, что интерфейсы слушают и IPv6 (наличие строки `inet6` в выводе):

```
$ ifconfig -a
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.101.5  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::42:c0ff:fea8:6505  prefixlen 64  scopeid 0x20<link>
        ether 02:42:c0:a8:65:05  txqueuelen 0  (Ethernet)
        RX packets 15787641  bytes 2553216408 (2.3 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9782965  bytes 1318487919 (1.2 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 605949  bytes 67799887 (64.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 605949  bytes 67799887 (64.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Когда я сталкиваюсь с непонятным поведением, то сразу начинаю яростно `strace`'ить. Так и сделал в этот раз.
Чтобы не цепляться к процессу nginx я его остановил и запустил их в паре.

```
$ nginx -s stop
$ strace -t -f  nginx -g 'daemon off;'
```

*Важно!* strace должен запускаться с опцией `-f`, в противном случае не увидим вывода worker'а nginx. Вот вырезка из
man:

>> -f
   Trace child processes as they are created by currently traced processes as a result of the fork(2), vfork(2) and clone(2) system calls. Note that -p PID -f will attach all threads of process PID if it is multi-threaded, not only thread with thread_id = PID.
 
В выводе сразу видны

```
10:24:47 open("/etc/host.conf", O_RDONLY|O_CLOEXEC) = 5
10:24:47 fstat(5, {st_mode=S_IFREG|0644, st_size=9, ...}) = 0
10:24:47 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f94b68fb000
10:24:47 read(5, "multi on\n", 4096)    = 9
10:24:47 read(5, "", 4096)              = 0
10:24:47 close(5)                       = 0

10:24:47 open("/etc/resolv.conf", O_RDONLY|O_CLOEXEC) = 5
10:24:47 fstat(5, {st_mode=S_IFREG|0644, st_size=38, ...}) = 0
10:24:47 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f94b68fb000
10:24:47 read(5, "nameserver 127.0.0.11\noptions nd"..., 4096) = 38
10:24:47 read(5, "", 4096)              = 0
10:24:47 close(5)                       = 0

10:24:47 open("/etc/hosts", O_RDONLY|O_CLOEXEC) = 5
10:24:47 fstat(5, {st_mode=S_IFREG|0644, st_size=202, ...}) = 0
10:24:47 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f94b68fb000
10:24:47 read(5, "127.0.0.1\tlocalhost\n::1\tlocalhos"..., 4096) = 202
10:24:47 read(5, "", 4096)              = 0
10:24:47 close(5)                       = 0

10:24:47 open("/etc/gai.conf", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
10:24:47 futex(0x7f94b5320420, FUTEX_WAKE_PRIVATE, 2147483647) = 0
10:24:47 socket(AF_INET, SOCK_DGRAM, IPPROTO_IP) = 5
10:24:47 connect(5, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
10:24:47 getsockname(5, {sa_family=AF_INET, sin_port=htons(52285), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
10:24:47 close(5)                       = 0

10:24:47 socket(AF_INET6, SOCK_DGRAM, IPPROTO_IP) = 5
10:24:47 connect(5, {sa_family=AF_INET6, sin6_port=htons(0), inet_pton(AF_INET6, "::1", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, 28) = 0
10:24:47 getsockname(5, {sa_family=AF_INET6, sin6_port=htons(57409), inet_pton(AF_INET6, "::1", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, [28]) = 0
10:24:47 close(5)                       = 0

```

