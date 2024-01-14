---
layout: post
title: "PHP-FPM: введение и структура конфигов"
image: /img/previews/php-fpm.png
tags: [php-fpm]
---

В данном посте собраны сведения по работе PHP-FPM, принцип его работы и описание конфигурирования.

FPM (FastCGI Process Manager) - это реализация FastCGI для PHP.

Интерфейс FastCGI — это клиент-серверный протокол взаимодействия веб-сервера и приложения.

Если обозначать схематически, то веб-сервер принимающий запросы пользователя, например `nginx`, преобразовывает
HTTP запросы в определенный набор данных (реализует протокол FastCGI) и передает эти данные процессу PHP-FPM.
PHP-FPM тоже реализовывает протокол FastCGI и может понять что от него хочет сервер. Далее PHP-FPM передает управление
интерпретатору PHP, а интерпретатор запускает пользовательский PHP скрипт.

![PHP-FPM Flow]({{ "/img/php-fpm-config-structure/php-fpm-flow.jpg" | absolute_url }}){: .in-post-img-center}

> PHP-FPM не реализовывает веб-сервер и не может принимать запросы от пользователя напрямую.

## Данные отправляемые по FastCGI протоколу

Так выглядит содержимое перехваченных пакетов сниффером [wireshark](https://www.wireshark.org/). Это HTTP запрос и его
обработал nginx.

![Request to nginx]({{ "/img/php-fpm-config-structure/php-fpm-request-to-nginx.jpg" | absolute_url }}){: .in-post-img-center .in-post-img-border}

nginx же отправляет следующие данные PHP-FPM процессу по FastCGI протоколу.

![Request from nginx to php-fpm]({{ "/img/php-fpm-config-structure/php-fpm-request-to-php-fpm.jpg" | absolute_url }}){: .in-post-img-center .in-post-img-border}

Ниже тот же самый запрос, но уже в более удобном для человека декодированном виде.

![Request from nginx to php-fpm decoded]({{ "/img/php-fpm-config-structure/php-fpm-request-to-php-fpm-decoded.jpg" | absolute_url }}){: .in-post-img-center .in-post-img-border}

### Мультиплексирование

Согласно [спецификации протокола](https://fastcgi-archives.github.io/FastCGI_Specification.html),
FastCGI клиент и FastCGI сервер могут использовать одно соединение для передачи данных независимых запросов. Также
можно использовать одно соединение для передачи нескольких потоков данных.

Если в классическом приложении всегда есть стандартные потоки ввода-вывода (stdout, stderr) и это два независимых
потока, то в FastCGI есть понятие record type, которое
[может иметь тип](https://fastcgi-archives.github.io/FastCGI_Specification.html#S5.3) `FCGI_STDIN`,
`FCGI_STDOUT` и `FCGI_STDERR`.

![FastGGI record types and flow]({{ "/img/php-fpm-config-structure/fastcgi-record-types-and-flow.jpg" | absolute_url }}){: .in-post-img-center}

В wireshark декодированные данные выглядят следующим образом.

![FastGGI record types and flow, wireshark]({{ "/img/php-fpm-config-structure/fastcgi-record-types-and-flow-wireshark.jpg" | absolute_url }}){: .in-post-img-center .in-post-img-border}

Таким образом, PHP-FPM обрабатывает независимо:

* `stdout`
* `stderr`
* `FCGI_STDOUT` и `FCGI_STDERR` по одному FastCGI соединению.

> В статье про [PHP-FPM logging]({{ site.baseurl }}{% link _posts/2022-05-08-php-fpm-logging-settings.md %}) детально
> рассматриваются аспекты логирования и объясняется почему эти отличия важны.

## master и child процессы<a name="master-child"></a>

PHP-FPM приложение запускает master процесс, основной задачей которого является запуск дочерних (child) процессов,
контроль за ними, периодический перезапуск и корректную остановку. Master процесс также следит за стандартными
потоками ввода-вывода stdin, stdout и stderr child процессов и может писать в них данные или читать из них данные.

Ниже пример запущенного PHP-FPM приложения с master и child процессами.

```shell
$ ps
PID   USER     TIME  COMMAND
    1 root      0:27 php-fpm: master process (/usr/local/etc/php-fpm.conf)
  267 www-data  0:00 php-fpm: pool www
  268 www-data  0:00 php-fpm: pool www
```

> Стоит отметить, что в примере master процесс запущен от пользователя root, а child процессы от пользователя
> www-data. Эта деталь важна, так как часть данных в логи может писать master процесс от одного пользователя, а
> часть данных будет писать child. Так вот, child процесс может не иметь прав на запись и при этом PHP-FPM никак
> не обозначит это. Просто не будет логов.

Master процесс не обрабатывает пользовательские PHP скрипты: обработкой занимаются child процессы. Если child процесс
умрет по какой-либо причине, то и master процесс останется жив, и другие child процессы не будут затронуты,
ну а master процесс запустит новый child процесс.

## Организация php-fpm конфига

PHP-FPM может использовать дефолтное значение пути конфиг файла. В примере выше, прямо в выводе команды `ps`,
видно какой конфигурационный файл использовался. Дефолтный путь можно получить выполнив команду `php-fpm -t`:

```shell
$ php-fpm -t
[18-Jun-2022 19:56:36] NOTICE: configuration file /usr/local/etc/php-fpm.conf test is successful
```

При запуске PHP-FPM можно указать путь к конфигу явно (например, `php-fpm -y /etc/php-fpm.conf`).

В каждом конфигурационном файле обязательно должна присутствовать секция `[global]` и хотя бы одна секция с
настройками группы child процессов для обработки пользовательских скриптов. В терминологии PHP-FPM подобная группа
называется _pool_.

Ниже приведен простой пример конфига `/usr/local/etc/php-fpm.conf`.

```ini
[global]
pid = /usr/local/var/run/php-fpm.pid
error_log = /usr/local/var/log/php-fpm.log
daemonize = no

[www]
user = www-data
group = www-data
listen = 9000
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

В данном конфиге объявлен pool `www`, который слушает запросы от веб-сервера (например, nginx) по TCP на порту 9000.
Пулов можно задать несколько.

### Использование `include`

Для упрощения администрирования и разделения конфигураций разных пулов применяется команда `include` позволяющая
подставить в месте вызова данные из другого файла. Обычно создается директория вроде `/usr/local/etc/php-fpm.d/`
и в нее помещаются файлы с настройками конкретных пулов.

**Пример.** `/usr/local/etc/php-fpm.conf` модифицируем следующим образом (порядок директив взят из [стандартных файлов
поставки `php-fpm`](https://github.com/php/php-src/blob/master/sapi/fpm/php-fpm.conf.in), о том что это небезопасно
читайте ниже).

```ini
[global]
pid = /usr/local/var/run/php-fpm.pid
error_log = /usr/local/var/log/php-fpm.log
daemonize = no

include = /usr/local/etc/php-fpm.d/*.conf
```

Ну а контент `www` пула перенесем в `/usr/local/etc/php-fpm.d/www.conf`.

```ini
[www]
user = www-data
group = www-data
listen = 9000
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

Директиву `include` можно использовать и внутри настроек пулов, например в `/usr/local/etc/php-fpm.d/www.conf`).
Данный подход может быть удобен в случае наличия нескольких пулов у которых должны быть общие настройки.
Продемонстрирую данный подход. Модифицируем `/usr/local/etc/php-fpm.d/www.conf`.

```ini
[www]
include = /usr/local/etc/php-fpm-common-pool.conf

user = www-data
group = www-data
listen = 9000
```

Создаем `/usr/local/etc/php-fpm-common-pool.conf` со следующим контентом.

```ini
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

Файл с общими настройками пула находится в одной директории с `php-fpm.conf`, а не в одной директории с `www.conf`.

```text
# tree
.
├── php-fpm-common-pool.conf
├── php-fpm.conf
└── php-fpm.d
    └── www.conf
```

**Почему не в директории `php-fpm.d`?**
PHP-FPM читает главный конфиг `php-fpm.conf` и выполняя директиву `icnlude` подставляет в то место где находится `include`
содержимое всех файлов объявленных в `php-fpm.conf`. При указании в директиве `include` glob паттерна `*.conf`
(`include = /usr/local/etc/php-fpm.d/*.conf`) содержимое всех файлов соответствующих паттерну будет помещено в место
объявления директивы `include`, т.е. на один уровень с секцией `[global]`. Настройки вроде
`pm, pm.max_children, pm.start_servers` и т.п. применимы только к пулам, потому PHP-FPM укажет, что такой
получившийся конфиг не валиден. Ниже иллюстрация каким бы получился конфиг после неверной подстановки:

```ini
[global]
pid = /usr/local/var/run/php-fpm.pid
error_log = /usr/local/var/log/php-fpm.log
daemonize = no

pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3

[www]
user = www-data
group = www-data
listen = 9000
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

Это явно не то что мы ожидали бы.

### Позиция `include` в конфигах

Конфиг `php-fpm.conf` обязан удовлетворять [`INI` формату](https://en.wikipedia.org/wiki/INI_file), но `INI` формат
не специфицирует поведения в ряде случаев и оставляет выбор варианта реализации на конкретной реализации `INI` парсера.

`INI` парсер PHP-FPM позволяет обрабатывать переопределение значений ключей, а также значений ключей в секциях.

> Если на базе примера пояснить, то `[global]` и `[www]` задают секции. В настройка `pm = dynamic`
> `pm` является ключом, а `dynamic` является значением. Настройка `pm = dynamic` помещена после объявления секции
> `[www]`, потому эта настройка применяется к секции `[www]`.

Таким образом, если мы поместим в `www.conf` дубли ключей, например, `listen`, то будет использовано последнее
установленное значение для ключа. Для секции `www` в примере ниже будет установлено значение `listen` равное `9999`.

```ini
[www]
user = www-data
group = www-data
listen = 9000
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3

; redefined
listen = 9999
```

Более того, из файла `www.conf` можно переопределить настройки секции `global`.

```ini
[global]
pid = /tmp/php-fpm.pid

[www]
user = www-data
group = www-data
listen = 9000
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

> Такая работа парсера открывает возможности для злоумышленников на shared хостингах, так что будьте внимательны.

Настройки одного пула могут переопределять настройки другого пула. Задание секции `[global]` в настройках пула
переопределяет настройки в главном конфиге `php-fpm.conf`.

По этой причине, место в файле где объявлен `include` играет важную роль.

```ini
[www]
include = /usr/local/etc/php-fpm-common-pool.conf

user = www-data
group = www-data
listen = 9000
```

В конфиге выше любые кастомизации значений в конфиге пула будут иметь преимущество над значениями заданными в
`/usr/local/etc/php-fpm-common-pool.conf`.

```ini
[www]
user = www-data
group = www-data
listen = 9000

include = /usr/local/etc/php-fpm-common-pool.conf
```

При таком конфиге последнее слово будет иметь настройка в `/usr/local/etc/php-fpm-common-pool.conf`.

С точки зрения же безопасности, то я рекомендую помещать в `php-fpm.conf` секцию с директивой `include` в самое начало.
Если используете официальный Docker image php-fpm, то будьте внимательны ([ниже пояснение](#docker-issues)).

### Порядок подключения `*.conf` файлов<a name="include-order"></a>

Как уже выяснили в предыдущем разделе, при варианте `include` сразу всех конфигов из директории
(`include=/usr/local/etc/php-fpm.d/*.conf`) важен их порядок подключения, ибо при
["грязном" подходе](#docker-issues) есть риск поломать PHP-FPM или получить неверную конфигурацию.

Что же касается рекомендаций, то если уже используете такие "грязные" хаки, то лучше называть конфиг файлы используя
числовые префиксы. PHP-FPM использует [`glob` функцию](https://linux.die.net/man/3/glob) для поиска именам файлов
для подстановки и `glob` по-умолчанию сортирует возвращаемый результат. Ниже пример именования и пример как будет
отсортирован результата `glob`.

```
/usr/local/etc/php-fpm.d/10-docker.conf
/usr/local/etc/php-fpm.d/50-aaa.conf
/usr/local/etc/php-fpm.d/50-www.conf
/usr/local/etc/php-fpm.d/99-docker.conf
```

## Несколько слов о Docker image официального PHP-FPM и его проблемах<a name="docker-issues"></a>

В стандартном [Dockerfile PHP-FPM](https://github.com/docker-library/php/blob/master/7.4/alpine3.16/fpm/Dockerfile#L215)
не уделяется внимание внутреннему устройству своих конфигов, а также делается ряд допущений, которые не прояснены.

**В образе нет разделения на `global` настройки и на настройки пулов**. Вообще нет упоминаний как настраивать
PHP-FPM. 

Структура конфигов выглядит следующим образом.

```
# tree /usr/local/etc
/usr/local/etc
├── pear.conf
├── php
│    ├── conf.d
│    │    └── docker-php-ext-sodium.ini
│    ├── php.ini-development
│    └── php.ini-production
├── php-fpm.conf
├── php-fpm.conf.default
└── php-fpm.d
    ├── docker.conf
    ├── www.conf
    ├── www.conf.default
    └── zz-docker.conf

3 directories, 10 files
```

`/usr/local/etc/php-fpm.conf` не содержит настроек, будут использоваться настройки по-умолчанию. В самом же конце
файла содержится директива `include`:

```ini
; Include one or more files. If glob(3) exists, it is used to include a bunch of
; files from a glob(3) pattern. This directive can be used everywhere in the
; file.
; Relative path can also be used. They will be prefixed by:
;  - the global prefix if it's been set (-p argument)
;  - /usr/local otherwise
include=etc/php-fpm.d/*.conf
```

Таким образом будет произведена подстановка значений конфигов, в указанном ниже порядке:
```
/usr/local/etc/php-fpm.d/docker.conf
/usr/local/etc/php-fpm.d/www.conf
/usr/local/etc/php-fpm.d/zz-docker.conf
```

Содержимое каждого из файлов:

```ini
; /usr/local/etc/php-fpm.d/docker.conf

[global]
error_log = /proc/self/fd/2

; https://github.com/docker-library/php/pull/725#issuecomment-443540114
log_limit = 8192

[www]
; if we send this to /proc/self/fd/1, it never appears
access.log = /proc/self/fd/2

clear_env = no

; Ensure worker stdout and stderr are sent to the main error log.
catch_workers_output = yes
decorate_workers_output = no
```

Из содержимого настроек пула `www` были убраны все закомментированные настройки.

```ini
; /usr/local/etc/php-fpm.d/www.conf

[www]
user = www-data
group = www-data
listen = 127.0.0.1:9000
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

Ну и "последний" в дефолтном списке:

```ini
[global]
daemonize = no

[www]
listen = 9000
```

**Какие тут проблемы?**

Во-первых, из настроек пула меняются настройки секции `[global]`. Непонятно зачем это сделано, но я настоятельно
рекомендую выносить явным образом настройки секции `[global]` именно в главный конфиг `/usr/local/etc/php-fpm.conf`.
Изменение настроек из пула - это неверная и опасная практика.

Во-вторых, вместо настроек пула `www` в одном месте они разнесены по разным файлам, что создает недоразумения если
лишь один пул `www` был модифицирован в вашем кастомном образе. Мол, а почему `listen = 8000` не имеет эффекта?

В-третьих, делается ложное предположение, что имя `zz-docker.conf` будет последним в списке файлов (`glob * pattern`),
а `docker.conf` - первым. Стоит только назвать иначе файл и не будет нарушен порядок и поведение, если
переопределялись настройки в кастомном конфиге пула.

## Что дальше?

Дальше рекомендую ознакомиться с детальным разбором
[механизмов логирование PHP-FPM]({{ site.baseurl }}{% link _posts/2022-05-08-php-fpm-logging-settings.md %}).
