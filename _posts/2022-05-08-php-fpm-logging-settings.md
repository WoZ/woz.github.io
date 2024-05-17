---
layout: post
title: PHP-FPM logging
image: /img/previews/php-fpm-logs.png
tags: [php-fpm]
---

С массовым приходом докеризации возникают новые сложности с которыми не приходилось работать раньше.
Одна такая сложность — это реализация подхода логирования согласно
[The twelve-factor app](https://12factor.net/logs) для PHP приложения в Docker контейнере. Другая сложность - это
правильная настройка PHP-FPM, PHP и nginx, так как в определенных случаях ошибки могут быть проброшены до nginx,
а это явно не то что ожидается. В данном посте собраны пояснения деталей работы всех возможных настроек PHP-FPM
и их хитросплетения. После прочтения у тебя точно не останется "серых" и непонятных зон работы
PHP-FPM, PHP и nginx в части логирования и обработки ошибок.

12-факторное приложение явно постулирует, что приложение не должно писать логи в лог-файлы на диске.
Вместо этого запись должна вестись в stdout. С PHP-FPM не все так гладко и одних лишь вариантов куда могут
попасть логи целое множество:

* PHP-FPM сам пишет логи, например о запуске master процесса PHP-FPM и своих child процессов;
* у PHP-FPM pool'а есть настройки для записи в access log;
* также для pool можно включить slowlog и фиксировать запросы выполняющиеся дольше положенного;
* настройка `catch_workers_output` включает и выключает перенаправление stdout/stderr PHP скрипта выполняющегося в
PHP-FPM child процессе;
* само PHP приложение может писать логи;
* а может и завершиться с fatal error;
* включенная настройка `fastcgi.logging` может перенаправить fatal errors и errors от PHP приложения к веб-серверу;
* а еще можно запустить PHP-FPM в демонизированном виде, либо опцией `--force-stderr` заставить писать в stderr.

![PHP-FPM and PHP settings]({{ "/img/php-fpm-logging-settings/php-fpm-and-php-settings.jpg" | absolute_url }}){: .in-post-img-center}

> В данном посте не будет идти речь о логировании в syslog, речь только о логировании в файлы или стандартные потоки
> ввода-вывода.

# Стандартные потоки ввода-вывода<a name="standard-streams"></a>

Прежде чем перейти к разбору кода PHP-FPM, стоит прояснить что такое стандартные потоки ввода-вывода
и как они функционируют. Даже если ты уже владеешь этими знаниями, настоятельно рекомендую освежить их в памяти.

У каждого запускаемого приложения есть 3 стандартных потока ввода/вывода
([standard streams](https://en.wikipedia.org/wiki/Standard_streams)):

* [stdin](http://www.linfo.org/standard_input.html) для чтения приложением входящих данных (ввод данных);
* [stdout](http://www.linfo.org/standard_output.html) для записи приложением данных, например, результатов выполнения;
* [stderr](http://www.linfo.org/standard_error.html) для записи приложением текста ошибок во время выполнения.

Работа с потоками ввода-вывода происходит путем чтения или записи из
[файлового дескриптора](https://en.wikipedia.org/wiki/File_descriptor). Файловый дескриптор - это идентификатор некоего открытого
файла, ну а поскольку в Linux файлами является почти все, то файловый дескриптор может указывать на привычный файл
на диске, а может указывать и на сокет сетевого соединения. У приложения есть просто номер дескриптора (обычный integer)
и оно делает `read` из дескриптора или `write` в дескриптор. Запущенное приложение всегда имеет файловые дескрипторы
с номерами 0, 1 и 2 ассоциированные с stdin, stdout и stderr соответственно.

Следовательно, **у приложения нет ни малейшего понятия ни о том откуда оно читает данные из stdin,
ни о конечном месте куда попадут данные записанные в stdout или stderr**. Например, когда в unix shell мы запускаем
приложение с перенаправлением ввода-вывода...

```shell
$ /usr/local/bin/my_command < /tmp/input.txt > /tmp/output.txt 2> /tmp/errors.txt
```

... мы указываем unix shell открыть файлы `/tmp/input.txt`, `/tmp/output.txt` и `/tmp/errors.txt`, а затем ассоциировать
их с соответствующими файловыми дескрипторами stdin, stdout и stderr для приложения `my_command`.

![stdio redirects]({{ "/img/php-fpm-logging-settings/stdio-redirects.jpg" | absolute_url }}){: .in-post-img-center}

Если запускать приложение без перенаправления вывода в терминале, то вывод stdout и stderr
будет производиться на экран с запущенным терминалом (не вдаваясь в подробности как до этого экрана вывод доедет, ведь
это может быть и терминал на удаленном сервере).

Что приложение может сделать **со своими** файловыми дескрипторами **стандартных потоков** ввода-вывода?

**Закрыть** (в языке `c` функцией [close](https://linux.die.net/man/2/close).
Возможности читать/писать больше не будет, при этом закрытие дескриптора 0, 1, 2 не приведет к
закрытию ассоциированных файлов или соединений, так как сами файлы открывались не приложением, а тем кодом, который
это приложение запускал, например, `bash`, `supervisord`, `cron`, [docker init](https://github.com/krallin/tini). 

Зачем это делать? Например, если не желает приложение больше читать или писать в поток.

<a name="standard-stream-dup"></a>
**Переопределить** (в языке `c` функциями [dup, dup2, dup3](https://linux.die.net/man/2/dup)).
Что это дает? Например, у приложения есть запись в stdout во множестве мест, без каких-либо специальных
функций-врапперов.

Приложение хочет проигнорировать перенаправление stdout в файл 
(`/usr/local/bin/my_command > /tmp/output.txt`) и хочет всегда писать в `/tmp/my_file.txt` вместо этого.

Чтобы не рефакторить код, можно открыть файл, а затем "привязать" новый дескриптор к дескриптору 1
(предопределенная константа STDOUT_FILENO равна 1). При открытии файла будет возвращен новый файловый дескриптор,
но после `dup2` оба дескриптора будут взаимозаменяемыми, а запись в дескриптор 1 будет эквивалентна записи в
дескриптор нового открытого файлы.

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

void main()
{
    int fd = open("/tmp/my_file.txt", O_RDWR | O_CREAT | O_APPEND);
    dup2(fd, STDOUT_FILENO); // STDOUT_FILENO is a constant, equal to 1 (stdout)
    close(fd);

    printf("%s\n", "hello world!");
    // or
    write(STDOUT_FILENO, "hello world!\n", 13);
}
```

Код выше демонстрирует этот подход: открывает файл, затем ассоциирует дескриптор 1 с этим файлом,
после чего закрывает файл. Покуда существуют хотя бы один ассоциированный дескриптор с файлом, то файл не
будет закрыт для записи или чтения. Потому дескриптор `fd` можно безопасно закрыть. Строки "hello world!" будут
записаны не в окно терминала, а в файл `/tmp/my_file.txt`, хотя в коде явно вызывается запись в STDOUT_FILENO.

Если тема с потоками ввода-вывода стала интересной, то рекомендую для продолжения исследования
[статью от Sichong Peng](http://www.sichong.site/linux/2021/11/06/dev-stdout-vs-2-a-dive-into-linux-stdio.html).

# Хитросплетения PHP-FPM

> Первым делом стоит уяснить для себя, что конфиги PHP-FPM почти не имеют ничего общего с конфигами самого PHP.
> 
> Я ранее уже писал про
[структуру конфигов PHP-FPM]({{ site.baseurl }}{% link _posts/2022-05-08-php-fpm-config-structure.md %}). Формулировка
"почти" присутствует по той причине, что в конфиге PHP имеется опция `fastcgi.logging`, которая применима только к
> реализациям SAPI FastCGI (например, PHP-FPM), но не настраивается на уровне PHP-FPM.

Введем термин **глобальный лог PHP-FPM**, либо просто **глобальный лог** который понадобиться для дальнейшего
разбора логики и объяснений.

**Глобальный лог PHP-FPM** (или сокращенно **глобальный лог** - это лог в который производится запись всех
возможных сообщений master процесса, а также сообщений от дочерних процессов при определенных настройках.
*Глобальный лог* может быть записан в stderr, либо в файл указанный в глобальном конфиге
[PHP-FPM error_log](https://www.php.net/manual/en/install.fpm.configuration.php#error-log).

В *глобальный лог* могут попадать и записи сделанные в
[access.log](https://www.php.net/manual/en/install.fpm.configuration.php#access-log) и
[slowlog](https://www.php.net/manual/en/install.fpm.configuration.php#slowlog), если специальным образом произвести
настройку. Это будет разъяснено позже.

В ряде случаев PHP-FPM проигнорирует значение из `error_log` и будет писать глобальный лог в другое место. По этой
причине и введен термин *глобальный лог*, так как говорить "логи пишутся в error_log" будет некорректно.
А может и не в error_log...

Примеры прямой записи в *глобальный лог*:

```shell
[02-Jul-2022 06:02:03] NOTICE: fpm is running, pid 1
[02-Jul-2022 06:02:03] NOTICE: ready to handle connections
```

[**access.log**](https://www.php.net/manual/en/install.fpm.configuration.php#access-log) - лог для фиксации факта
выполненного запроса
[child процессом PHP-FPM]({{ site.baseurl }}{% link _posts/2022-05-08-php-fpm-config-structure.md %}#master-child),
позволяет записать дату запроса, запрашиваемый URL, статус код ответа, а также время выполнения, потребленную память
во время запроса и т.п. Задается для каждого пула отдельно в одноименной настройке пула
*access.log*.

Пример записей в *access.log*:

```shell
172.16.6.3 -  18/Jun/2022:07:32:15 +0000 "GET /index.php" 404
172.16.6.3 -  18/Jun/2022:07:45:39 +0000 "GET /phpinfo.php" 200
172.16.6.3 -  18/Jun/2022:07:46:22 +0000 "GET /error_handling/default_configs.php" 200
```

[**slowlog**](https://www.php.net/manual/en/install.fpm.configuration.php#slowlog) - лог для записи stacktrace
PHP скрипта в момент превышении им времени выполнения заданного в настройке пула PHP-FPM
[request_slowlog_timeout](https://www.php.net/manual/en/install.fpm.configuration.php#request-slowlog-timeout).
*slowlog* является тоже настройкой пула PHP-FPM.

При включенной настройке `request_slowlog_timeout`, master процесс PHP-FPM начинает контролировать время
выполнения PHP скриптов в child процессах. Когда время выполнения превысит значение `request_slowlog_timeout`,
то PHP-FPM залогирует в *глобальный лог* факт длительного выполнения и попытается получить stacktrace для
записи в *slowlog*.

Пример записей в глобальном логе:

```shell
[11-Jul-2022 05:34:02] WARNING: [pool www] child 155, script '/var/www/html/public/error_handling/slowlog.php' (request: "GET /error_handling/slowlog.php") executing too slow (1.176006 sec), logging
[11-Jul-2022 05:34:02] NOTICE: child 155 stopped for tracing
[11-Jul-2022 05:34:02] NOTICE: about to trace 155
[11-Jul-2022 05:34:02] NOTICE: finished trace of 155
```

Пример записей в *slowlog*:

```shell
[11-Jul-2022 05:34:02]  [pool www] pid 155
script_filename = /var/www/html/public/error_handling/slowlog.php
[0x00007f63028160f0] sleep() /var/www/html/public/error_handling/slowlog.php:13

[11-Jul-2022 05:36:59]  [pool www] pid 156
script_filename = /var/www/html/public/index.php
[0x00007f63028161b0] mysqlConnect() /var/www/html/public/index.php:18
[0x00007f6302816150] dbConnect() /var/www/html/public/index.php:22
[0x00007f63028160f0] main() /var/www/html/public/index.php:25
```

Если помимо `request_slowlog_timeout` включить настройку
[request_terminate_timeout](https://www.php.net/manual/en/install.fpm.configuration.php#request-terminate-timeout),
то child процесс будет аварийно остановлен при превышении `request_terminate_timeout`, а в глобальный лог
записан факт остановки процесса и запуска нового:

```shell
[11-Jul-2022 05:41:25] WARNING: [pool www] child 158, script '/var/www/html/public/index.php' (request: "GET /error_handling/index.php") executing too slow (1.308849 sec), logging
[11-Jul-2022 05:41:25] NOTICE: child 158 stopped for tracing
[11-Jul-2022 05:41:25] NOTICE: about to trace 158
[11-Jul-2022 05:41:25] NOTICE: finished trace of 158
[11-Jul-2022 05:41:26] WARNING: [pool www] child 158, script '/var/www/html/public/index.php' (request: "GET /index.php") execution timed out (2.314836 sec), terminating
[11-Jul-2022 05:41:26] WARNING: [pool www] child 158 exited on signal 15 (SIGTERM) after 6.016552 seconds from start
[11-Jul-2022 05:41:26] NOTICE: [pool www] child 160 started
```

Запись stacktrace может произойти только если включена CAP_SYS_PTRACE capability, либо выключены проверки вовсе
(`/proc/sys/kernel/yama/ptrace_scope`). Без включения будет примерно следующий вывод в глобальный лог:

```
ERROR: failed to ptrace(ATTACH) child 15: Operation not permitted
```

Для docker
[решением является](https://jvns.ca/blog/2020/04/29/why-strace-doesnt-work-in-docker/) включение
`--cap-add=SYS_PTRACE`. Но и
[злоумышленники могут использовать CAP_SYS_PTRACE](https://blog.pentesteracademy.com/privilege-escalation-by-abusing-sys-ptrace-linux-capability-f6e6ad2a59cc)
для эскалации прав.


## Подготовка со стороны master процесса

Когда PHP-FPM процесс запускается и еще не успел породить child процессы, то у master процесса уже есть
дескрипторы stdin, stdout и stderr. В разделе про [стандартные потоки ввода-вывода](#standard-streams) об этом детально
рассказано.

Ниже мы разберем логику ряда функций выполняемых при инициализации. Порядок их упоминания соответствует
порядку выполнения.

![php-fpm functions]({{ "/img/php-fpm-logging-settings/php-fpm-functions.jpg" | absolute_url }}){: .in-post-img-center}


* fpm_stdio_init_main - ассоциирует stdin и stdout master процесса с `/dev/null`;
* fpm_stdio_open_error_log - конфигурирует куда писать *глобальный лог* (stderr или значение из глобальной
настройки PHP-FPM `error_log`), открывает файлы при необходимости;
* fpm_log_open - конфигурирует запись в *access.log* и открывает файлы при необходимости;
* fpm_stdio_init_final - если используется PHP-FPM `error_log` файл для записи глобальных логов, то произойдет
ассоциация со stderr и оригинальный stderr дескриптор будет утерян.
* fpm_stdio_prepare_pipes - создает pipes для доставки сообщений от child процесса к master процессу.

Ниже в коде будет встречаться функция `zlog`. Разбор функции будет позже, но для дальнейшего понимания немного
информации о функции:

* в master процессе вызов `zlog` запишет строку в *глобальный лог*;
* в child процессе `zlog` всегда пишет в `stderr` child процесса, который *может быть* перенаправлен в master процессу
и тоже можеть быть записан в *глобальный лог*.

### fpm_stdio_init_main

PHP-FPM в функции
[fpm_stdio_init_main](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L35)
открывает `/dev/null` и ассоциирует stdin и stdout с `/dev/null`. С этого момента все что будет записано master
процессом в stdout будет отправлено в специальный [null device](https://en.wikipedia.org/wiki/Null_device), который
удаляет все данные записанные в него. Проще говоря, с этого момента весь output в stdout уйдет в никуда.
При чтении null device соответствует пустому файлу без содержимого.

```c
int fpm_stdio_init_main()
{
    int fd = open("/dev/null", O_RDWR);

    // ...

    if (0 > dup2(fd, STDIN_FILENO) || 0 > dup2(fd, STDOUT_FILENO)) {
        zlog(ZLOG_SYSERROR, "failed to init stdio: dup2()");
        close(fd);
        return -1;
    }
    close(fd);
    return 0;
}
```

![php-fpm stdin stdout to /dev/null]({{ "/img/php-fpm-logging-settings/php-fpm-stdin-stdout-dev-null.jpg" | absolute_url }}){: .in-post-img-center}

> На диаграмме выше и всех последующих диаграммах такого рода, направление стрелки с подписью *dup2* означает, что
дескриптор на который указывает стрелка будет закрыт и ассоциирован с дескриптором из которого стрелка выходит.

По этой причине бесполезно указывать для PHP-FPM конфигов `error_log`, `access.log` или `slowlog`
в качестве значений `/dev/stdout` или `/proc/self/fd/1`. Данные уйдут в никуда.

### fpm_stdio_open_error_log<a name="fpm-stdio-open-error-log"></a>

К этому моменту будут прочитаны конфигурационные файлы и получены значения конфигов PHP-FPM `error_log`,
`access.log` и `slowlog`.

Будет осуществлен вызов
[fpm_stdio_open_error_log](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L317)
с параметром 0 (reopen == 0).

```c
int fpm_stdio_open_error_log(int reopen)
{
    // syslog processing

    fd = open(fpm_global_config.error_log, O_WRONLY | O_APPEND | O_CREAT, S_IRUSR | S_IWUSR);
    // error processing...

    if (reopen) {
        // only executes when reopen == 1, skipped
    } else {
        fpm_globals.error_log_fd = fd;
        if (fpm_use_error_log()) {
            zlog_set_fd(fpm_globals.error_log_fd);
        }
    }
    
    // other actions
}
```

Нас интересует вызов
[fpm_use_error_log](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L45),
так как именно он определяет нужно ли использовать настройку PHP-FPM `error_log`, либо принудительно использовать
`stderr` для записи *глобального лога*.

```c
static inline int fpm_use_error_log(void) {
    /*
     * the error_log is NOT used when running in foreground
     * and from a tty (user looking at output).
     * So, error_log is used by
     * - SysV init launch php-fpm as a daemon
     * - Systemd launch php-fpm in foreground
     */
#ifdef HAVE_UNISTD_H
    if (fpm_global_config.daemonize || (!isatty(STDERR_FILENO) && !fpm_globals.force_stderr)) {
#else
    if (fpm_global_config.daemonize) {
#endif
        return 1;
    }
    return 0;
}
```

Для простоты понимания, ниже представлена диаграмма логики принятия решений `fpm_use_error_log`.

![fpm_use_error_log]({{ "/img/php-fpm-logging-settings/php-fpm-use-error-log.jpg" | absolute_url }}){: .in-post-img-center}

Сейчас же пройдемся по каждому пункту проверки.

#### daemonize or not daemonize, that is the question

У PHP-FPM есть параметры командной строки для включения/выключения демонизации, а также глобальная настройка
[daemonize](https://www.php.net/manual/en/install.fpm.configuration.php#daemonize). Настройки полностью эквивалентны,
но параметры командной строки имеют наивысший приоритет, потому рассмотрим их.

> Рекомендую использовать только настройки командной строки. В противном случае можно столкнуться с проблемами
> переопределения и порядка подключения конфигов PHP-FPM. Об этих проблемах я
> [писал в предыдущем посте]({{ site.baseurl }}{% link _posts/2022-05-08-php-fpm-config-structure.md %}#include-order).

```shell
$ php-fpm --help
...

-D, --daemonize  force to run in background, and ignore daemonize option from config file
-F, --nodaemonize force to stay in foreground, and ignore daemonize option from config file
```

Запуская PHP-FPM без демонизации (`php-fpm -F`) в shell (например, в bash), запущенный процесс не уходит в
background и в stderr заданный shell будет перенаправляться все что master процесс отправит в stderr.
`pid` master процесса не будет изменен.

С демонизацией (`php-fpm -D`) будет запущен master процесс PHP-FPM, который почти сразу запустит свою копию, но уже
отвязавшись от shell exec и в background, а старый master процесс завершиться.
Новый master процесс будет иметь новый `pid`.

Для docker контейнера разница с `pid` важна.

Во-первых, docker запускает команду/приложение указанную в CMD/ENTRYPOINT
секции и [при ее смерти останавливает контейнер](https://www.tutorialworks.com/why-containers-stop/). Запустив
PHP-FPM в демонизированном режиме docker увидит что процесс завершился и остановит контейнер. Не важно что
PHP-FPM перед этим создал новый процесс в background.

Во-вторых, docker привязывает стандартные потоки ввода-вывода команды/приложения к своему logging driver и читает
stdout с stderr запущенного процесса указанного в секции CMD/ENTRYPOINT. При завершении изначально порожденного
процесса docker не смог бы читать логи.

Да и функция `fpm_use_error_log` о которой шла речь выше, при включении демонизации вернет 1, чем заставит
открыть файл из настройки `error_log` и перенаправлять *глобальный лог* в `error_log` файл.

> Итого...
> Хотите запускать php-fpm в docker? Не используйте демонизацию!
> Не используете docker, но хотите видеть глобальный лог в stderr? Запускайте без демонизации.

#### isatty(STDERR_FILENO) и force-stderr

PHP-FPM можно запустить с опцией `--force-stderr`:

```
$ php-fpm --help
...
-O, --force-stderr force output to stderr in nodaemonize even if stderr is not a TTY
```

Запуск с `--force-stderr` устанавливает в 1 значение `fpm_globals.force_stderr`.

Вернемся снова к блоку кода `fpm_use_error_log`

```c
if (fpm_global_config.daemonize || (!isatty(STDERR_FILENO) && !fpm_globals.force_stderr)) {
    return 1;
}
return 0;
```

`isatty(STDERR_FILENO)` возвращает 1 если дескриптор stderr (`STDERR_FILENO`) является терминалом.
`STDERR_FILENO` будет терминалом если PHP-FPM запущен в терминале, но если stderr является файлом, либо PHP-FPM
запускается в Docker container, то будет возврат 0.

Запуск в терминале без перенаправления потоков и независимо от включения `--force-stderr` заставит
PHP-FPM писать *глобальный лог* в stderr.

#### Некоторые примеры

**Пример 1.** Перенаправление stderr в файл, запуск в терминале, без `--force-stderr`.

Конфиг PHP-FPM:

```ini
[global]
error_log = /usr/local/var/log/php-fpm.log
```

Команда запуска PHP-FPM:

```shell
$ php-fpm -F 2> /tmp/errors.txt
```

`isatty` вернет 0, а `--force-stderr` не включен, потому вывод будет направлен в файл `/usr/local/var/log/php-fpm.log`.

**Пример 2.** Перенаправление stderr в файл, запуск в терминале, но включена опция `--force-stderr`.

Конфиг PHP-FPM:

```ini
[global]
error_log = /usr/local/var/log/php-fpm.log
```

Команда запуска PHP-FPM:

```shell
$ php-fpm -F --force-stderr 2> /tmp/errors.txt
```

`fpm_use_error_log` вернет 0, *глобальный лог* будет записан в stderr, а затем перенаправлен в `/tmp/errors.txt`.
Как итог, *глобальный лог* будет записан в `/tmp/errors.txt`.

**Пример 3.** Запуск в Docker, без перенаправления, без `--force-stderr`.

Конфиг `php-fpm`:

```ini
[global]
error_log = /usr/local/var/log/php-fpm.log
```

Команда запуска `php-fpm`:

```
$ php-fpm -F
```

`isatty` вернет 0 так как запуск не в терминале. `fpm_globals.force_stderr` равен 0.
Функция `fpm_use_error_log` вернет 1 и *глобальный лог* будет записан в `/usr/local/var/log/php-fpm.log`.

**Пример 4.** Запуск в Docker, без перенаправления, с `--force-stderr`.

Конфиг PHP-FPM:

```ini
[global]
error_log = /usr/local/var/log/php-fpm.log
```

Команда запуска PHP-FPM:

```
$ php-fpm -F --force-stderr
```

`isatty` вернет 0 так как запуск не в терминале. `fpm_globals.force_stderr` равен 1.
Функция `fpm_use_error_log` вернет 0 и *глобальный лог* будет записан в stderr и попадет в docker logs.

**Пример 5.** Запуск в Docker, конфиг `error_log` задает запись в stderr, `--force-stderr` не включен.

> В официальном docker image php-fpm как раз используется такой подход, с которым я не согласен и считаю ошибочным.

```ini
[global]
error_log = /proc/self/fd/2
```

Команда запуска PHP-FPM:

```
$ php-fpm -F
```
`isatty` вернет 0 так как запуск не в терминале. `fpm_globals.force_stderr` равен 0. Функция `fpm_use_error_log`
вернет 1 и глобальный лог будет записан в `/proc/self/fd/2`.

Но `/proc/self/fd/2`
[является магическим указателем](https://unix.stackexchange.com/questions/676683/what-does-the-output-of-ll-proc-self-fd-from-ll-dev-fd-mean)
и запись в него имеет аналогичный эффект записи в stderr.

Docker читает stderr процесса PHP-FPM, потому *глобальный лог* будет досталвен в Docker и docker logs покажут данные
из лога. PHP-FPM при этом будет считать что работает с обычным файлом, собственно `fpm_use_error_log` и возвращает 1.

### fpm_log_open

Окей, после выполнение `fpm_stdio_open_error_log` и `fpm_use_error_log` мы имеем открытый `error_log`, конечно,
если он должен использоваться. Чтобы продемонстрировать расширенную логику, предполагаем что он использоваться будет.

![php-fpm master phase 1]({{ "/img/php-fpm-logging-settings/php-fpm-master-phase-1.jpg" | absolute_url }}){: .in-post-img-center}

Далее наступит черед вызова
[fpm_log_open](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_log.c#L31),
с параметром 0 (repoen == 0), которая проходит по всем пулам указанных в конфиге и открывает заданные в настройках
пулов `access.log` файлы.

```c
int fpm_log_open(int reopen)
{
    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        if (!wp->config->access_log) {
            continue;
        }

        // some code was skipped

        fd = open(wp->config->access_log, O_WRONLY | O_APPEND | O_CREAT, S_IRUSR | S_IWUSR);
        if (0 > fd) {
            zlog(ZLOG_SYSERROR, "failed to open access log (%s)", wp->config->access_log);
            return -1;
        } else {
            zlog(ZLOG_DEBUG, "open access log (%s)", wp->config->access_log);
        }

        if (reopen) {
            // reopen == 0, skipped
        } else {
            wp->log_fd = fd;
        }

        // some code was skipped

    return ret;
}
```

Таким образом, открытие `access.log` происходит в master процессе, а не в дочерних, но писать в `access.log` будут
исключительно child процессы (об этом ниже).

![php-fpm master phase 2]({{ "/img/php-fpm-logging-settings/php-fpm-master-phase-2.jpg" | absolute_url }}){: .in-post-img-center}


### fpm_stdio_init_final

Метод
[fpm_stdio_init_final](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L64)
использует подход с [переопределением](#standard-stream-dup) потока stderr.

```c
int fpm_stdio_init_final()
{
    if (fpm_use_error_log()) {
        /* prevent duping if logging to syslog */
        if (fpm_globals.error_log_fd > 0 && fpm_globals.error_log_fd != STDERR_FILENO) {

            /* there might be messages to stderr from other parts of the code, we need to log them all */
            if (0 > dup2(fpm_globals.error_log_fd, STDERR_FILENO)) {
                zlog(ZLOG_SYSERROR, "failed to init stdio: dup2()");
                return -1;
            }
        }
#ifdef HAVE_SYSLOG_H
        else if (fpm_globals.error_log_fd == ZLOG_SYSLOG) {
            /* dup to /dev/null when using syslog */
            dup2(STDOUT_FILENO, STDERR_FILENO);
        }
#endif
    }
    zlog_set_launched();
    return 0;
}
```

Важный момент - это вызов `dup2(fpm_globals.error_log_fd, STDERR_FILENO)`.

Таким образом, если `php-fpm` определился с использованием `error_log` для записи *глобального лога*, то
дескриптор stderr будет ассоциирован с дескриптором `fpm_globals.error_log_fd`,
который является дескриптором открытого файла указанного в `error_log` конфиге PHP-FPM.

Если теперь после произведенной ассоциации какая либо логика PHP-FPM **попытается открыть** файл `/dev/stderr`, либо
`/proc/self/fd/2`, а затем **попробует записать** данные в этот **новый открытый дескриптор**, то эти данные
попадут не в stderr как ожидается, а в файл заданный в `error_log` конфиге PHP-FPM!

> А теперь вспомните, что *slowlog* открывается и закрывается каждый раз когда фиксируется превышение времени
> выполнения. Если конфиг `slowlog` установлен в `/dev/stdout` или `/proc/self/fd/2`, то *slowlog* будет записан
> в `error_log` файл. В stderr записи *slowlog* попадут лишь когда `error_log` установлен в `/dev/stdout` или
> `/proc/self/fd/2`.

### fpm_stdio_prepare_pipes<a name="fpm-stdio-prepare-pipes"></a>

Последняя функция перед вызовом fork связанная с логированием.

Функцией
[fpm_stdio_prepare_pipes](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L237)
для каждого создаваемого пула серверов происходит проверка установки конфига `catch_workers_output`.

Когда опция включена, то master еще до вызова fork создает пару [pipes](https://man7.org/linux/man-pages/man2/pipe.2.html)
для обычного output (fd_stdout) и ошибок (fd_stderr).

Pipe представляет собой пару дескрипторов один из которых является дескриптором куда можно писать (write end),
а другой дескриптор предназначен для чтения записанного контента (read end). Такая себе труба где в один конец
информация влетает, а из другого информацию можно прочесть.

Эти пайпы нужны для передачи сообщений и логов от child процесса.

Но если настройка `catch_workers_output` не включена, то пайпы не создаются и в дальнейшем использоваться не будут.
Так что доставки логов и сообщений от child процессов до master процесса тоже не будет.

## Порождение child процессов

После прохождения предыдущих шагов у нас все еще запущен master процесс, но он уже готов создавать дочерние.

![php-fpm master phase 1]({{ "/img/php-fpm-logging-settings/php-fpm-master-phase-2.jpg" | absolute_url }}){: .in-post-img-center}

Сhild процессы порождаются вызовом функции [fork](https://man7.org/linux/man-pages/man2/fork.2.html), которая создает
дубликат вызывающего процесса (master) и каждый из процессов продолжает свое выполнение со следующей инструкции
независимо друг от друга.

![php-fpm functions after fork]({{ "/img/php-fpm-logging-settings/php-fpm-functions-after-fork.jpg" | absolute_url }}){: .in-post-img-center}

Все файловые дескрипторы открытые до вызова master процессом fork будут доступны и child процессу.
Сhild может читать и писать в эти унаследованные файловые дескрипторы.

![php-fpm master and child phase 2]({{ "/img/php-fpm-logging-settings/php-fpm-master-child-phase-2.jpg" | absolute_url }}){: .in-post-img-center}

### fpm_stdio_parent_use_pipes

[Эта функция](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L267)
будет выполнена только в master процессе. Если настройка `catch_worker_output` была включена, то функция
[fpm_stdio_prepare_pipes](#fpm-stdio-prepare-pipes) ранее создала пайпы для доставки stdout и stderr от child процессов
master процессу.

Задача данной функции состоит в закрытии write end пайпов `fd_stdout` и `fd_stderr` чтоб ненароком кто не записал
из master процесса туда данные.

Далее вызываются функции `fpm_event_set` и `fpm_event_add` запускающие механизм прослушивания сообщений доставляемых
в read end пайпов. Такая себе подписка на пришедшие события. Благодаря этой подписке можно получать как сообщения
об ошибках самого PHP-FPM child, так и данные записанные PHP скриптами выполняющимися в child процессе в stdout и
stderr.

```c
int fpm_stdio_parent_use_pipes(struct fpm_child_s *child)
{
    if (0 == child->wp->config->catch_workers_output) { /* not required */
        return 0;
    }

    close(fd_stdout[1]);
    close(fd_stderr[1]);

    child->fd_stdout = fd_stdout[0];
    child->fd_stderr = fd_stderr[0];

    fpm_event_set(&child->ev_stdout, child->fd_stdout, FPM_EV_READ, fpm_stdio_child_said, child);
    fpm_event_add(&child->ev_stdout, 0);

    fpm_event_set(&child->ev_stderr, child->fd_stderr, FPM_EV_READ, fpm_stdio_child_said, child);
    fpm_event_add(&child->ev_stderr, 0);
    return 0;
}
```

После `fpm_stdio_parent_use_pipes` запускается event loop и master выполняет функции контроля за child процессами,
контроля за сигналаи и перенаправления сообщений из пайпов в *глобальный лог*.

### fpm_stdio_child_use_pipes

Данная функция (как и последующие) выполняются только в child процессах.

Логика функции зависит от настройки `catch_workers_output`. По-умолчанию в `php-fpm` `catch_workers_output = no`, т.е.
выключена, а вот в официальном docker образе она включена.

Логика принятия решений находится в функции
[fpm_stdio_child_use_pipes](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L303).

#### catch_workers_output = no<a name="catch-workers-output-no"></a>

```c
void fpm_stdio_child_use_pipes(struct fpm_child_s *child)
{
    if (child->wp->config->catch_workers_output) {
        // see this part below
    } else {
        /* stdout of parent is always /dev/null */
        dup2(STDOUT_FILENO, STDERR_FILENO);
    }
}
```

![php-fpm catch_workers_output is off]({{ "/img/php-fpm-logging-settings/php-fpm-catch-workers-output-off.jpg" | absolute_url }}){: .in-post-img-center}

При выключенной настройке child процесс ассоциирует stderr с stdout, ну а поскольку ранее он еще master процессом
был ассоциирован с `/dev/null`, то попытка записи в поток stdout или stderr именно child процесса приведет к записи
в null device и эти записи исчезнут.

![php-fpm catch_workers_output is off part 2]({{ "/img/php-fpm-logging-settings/php-fpm-catch-workers-output-off-2.jpg" | absolute_url }}){: .in-post-img-center}

#### catch_workers_output = yes<a name="catch-workers-output-yes"></a>

Так как функция вызывается еще до fork, то у master и child есть дескрипторы и read, и write end. Помните про
особенности fork? Так вот, master процесс закрывает write end чтобы случайно разработчики PHP-FPM не добавили код
записывающий что нибудь в этот pipe.

> Напомню, что аналогичным образом поступает и child, но уже с read end, чтоб случайно никто читать не начал.

Когда же функция
[fpm_stdio_child_use_pipes](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L303)
вызывается с настройкой пула `catch_workers_output = yes`, то происходит ассоциация stdout и stderr child процесса
с write end созданных pipes!

```c
void fpm_stdio_child_use_pipes(struct fpm_child_s *child)
{
    if (child->wp->config->catch_workers_output) {
        dup2(fd_stdout[1], STDOUT_FILENO);
        dup2(fd_stderr[1], STDERR_FILENO);
        close(fd_stdout[0]); close(fd_stdout[1]);
        close(fd_stderr[0]); close(fd_stderr[1]);
    } else {
        // was discussed in the previous section
    }
}
```

![php-fpm catch_workers_output is on]({{ "/img/php-fpm-logging-settings/php-fpm-catch-workers-output-on.jpg" | absolute_url }}){: .in-post-img-center}

Запись в stdout и stderr теперь разделена и master процесс может отличить одно от другого.

![php-fpm catch_workers_output is on part 2]({{ "/img/php-fpm-logging-settings/php-fpm-catch-workers-output-on-2.jpg" | absolute_url }}){: .in-post-img-center}

С этого момента все ошибки child процесса, а также запись PHP скриптами в stdout и stderr будет доставлена master
процессу.

### fpm_stdio_init_child<a name="fpm-stdio-init-child"></a>

Следующим шагом у child процесса происходит закрытие дескриптора файла из конфига `error_log` функцией
[fpm_stdio_init_child](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/fpm_stdio.c#L88).

> Повторюсь, если `error_log`, а не stderr должен использоваться master процессом для записи *глобального лога*.

```c
int fpm_stdio_init_child(struct fpm_worker_pool_s *wp)
{
    // syslog processing

    if (fpm_globals.error_log_fd > 0) {
        close(fpm_globals.error_log_fd);
    }
    fpm_globals.error_log_fd = -1;
    zlog_set_fd(-1);

    return 0;
}
```

На иллюстрации ниже отмечены неиспользуемые или закрытые дескрипторы:

* `stdin` не используется и далее его можно не рассматривать;
* `error_log` файл не будет использован child процессом и закрыт;
* а вот состояние `stdout` и `stderr` уже обсудили выше.

![php-fpm master child phase 2 excluded]({{ "/img/php-fpm-logging-settings/php-fpm-master-child-phase-2-excluded.jpg" | absolute_url }}){: .in-post-img-center}

### fpm_log_init_child

Child процесс унаследовал файловый дескриптор `access.log` файла (если он был указан в конфигах, конечно же) и в функции
[fpm_log_init_child](https://github.com/php/php-src/blob/327bb219867b16e1161da632fba170f46692484b/sapi/fpm/fpm/fpm_log.c#L73)
происходит установка этого дескриптора для последующего использования.

Таким образом, информацию в `access.log` пишет child процесс используя унаследованный дескриптор от master процесса.

А что делать при наобходимости reload логфайлов? Ротацию будет производить master процесс, при этом child процессы
будут перезапущены, потому проблем с этим нет.

### zlog_set_external_logger<a name="zlog-set-external-logger"></a>

Последний вызов в цепочке

```c
zlog_set_external_logger(sapi_cgi_log_fastcgi);
```

```c
void zlog_set_external_logger(void (*logger)(int, char *, size_t))
{
    external_logger = logger;
}
```

Данная функция устанавливает функцию
[sapi_cgi_log_fastcgi](https://github.com/php/php-src/blob/b5db594fd277464104fce814d22f0b2207d6502d/sapi/fpm/fpm/fpm_main.c#L592)
в качестве особого обработчика.

Каждый раз когда происходит ошибка у child, fatal error у скрипта, либо же **`error_log` в php.ini** указывает на
`stderr` и происходит соответствующее событие, то эти сообщения попадают в функцию `vzlog`.

Ну и если в php.ini установлена настройка `fastcgi.logging`, то все перечисленные выше сообщения попадут в обработчик
`sapi_cgi_log_fastcgi` и будут доставлены веб-серверу.

> Довольно сомнительный функционал, потому рекомендую выключать и настраивать нормально логирование на уровне PHP.

![php-fpm fastcgi.logging]({{ "/img/php-fpm-logging-settings/php-fpm-fastcgi-logging.jpg" | absolute_url }}){: .in-post-img-center}

## Функционал записи в глобальный лог

Вот теперь подобрались к самому концу и можно рассмотреть как же попадают в *глобальный лог* записи:

* от master процесса;
* от child процессов;
* PHP скриптами в stdout и stderr (в child процессах);
* errors и fatal errors.

### zlog

В исходном коде PHP-FPM запись в лог осуществляется макросом
[`zlog`](https://github.com/php/php-src/blob/1c753a958bca9b3c802954bbb2d0681235e4af93/sapi/fpm/fpm/zlog.h#L9).
При этом не важно пишет master или child. Функция `zlog` используется всегда.

Вот некоторые примеры из кода вызова `zlog`

```c
zlog(ZLOG_NOTICE, "ready to handle connections");
zlog(ZLOG_ERROR, "FPM initialization failed");
zlog(ZLOG_SYSERROR, "failed to open access log (%s)", wp->config->access_log);
zlog(ZLOG_NOTICE, "child %d stopped for tracing", (int) pid);
```

Макрос `zlog` вызывает функцию
[zlog_ex](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/zlog.c#L254),
подмешивая название вызвавшей функции и строку откуда был сделан вызов.

`zlog_ex` далее вызывает функцию
[vzlog](https://github.com/php/php-src/blob/305d5e12dff31eddf43cee63675c7012cca10e43/sapi/fpm/fpm/zlog.c#L193).

Нас интересует код в функции `vzlog`

```c
void vzlog(const char *function, int line, int flags, const char *fmt, va_list args)
{
    // skipped a lot of code
    
#ifdef HAVE_SYSLOG_H
    if (zlog_fd == ZLOG_SYSLOG) {
        buf[len] = '\0';
        php_syslog(syslog_priorities[zlog_level], "%s", buf);
        buf[len++] = '\n';
    } else
#endif
    {
        buf[len++] = '\n';
        zend_quiet_write(zlog_fd > -1 ? zlog_fd : STDERR_FILENO, buf, len);
    }

    if (zlog_fd != STDERR_FILENO && zlog_fd != -1 &&
            !launched && (flags & ZLOG_LEVEL_MASK) >= ZLOG_NOTICE) {
        zend_quiet_write(STDERR_FILENO, buf, len);
    }
```

> [zend_quiet_write](https://github.com/php/php-src/blob/c1a06704daa97ff71781dab3ccd7ffa805dbdd9a/Zend/zend_portability.h#L129)
является макросом оборачиващим сишную функцию write.

> Код с проверкой `!launched` будет отработан лишь master процессом до fork PHP-FPM. Сделано это для возможности
> логирования ошибок инициализации в stderr в принудительном порядке.

Первая часть блока кода касается записи в syslog, которую мы в этом посте не рассматриваем.
А вот дальше идет блок кода проверяющий значение `zlog_fd`.

#### zlog у child процесса

Для child процесса после вызова `fpm_stdio_init_child`, [разбор которого был ранее](#fpm-stdio-init-child),
значение переменной `zlog_fd` равно -1. Это значение устанавливает функция `zlog_set_fd`, что была приведена в разборе
`fpm_stdio_init_child`.

Следовательно, все данные переданные в `zlog` в child процессе попадут в stderr.

Когда настройка `catch_workers_output` выключена, то [согласно разбору](#catch-workers-output-no), запись в `stderr`
пойдут в `/dev/null`.

При [включенной настройке](#catch-workers-output-yes) `catch_workers_output` данные будут доставлены master процессу.

#### zlog у master процесса

В методе `fpm_stdio_open_error_log` ([разбор](#fpm-stdio-open-error-log)) в качестве *глобального лога* может быть
выбран либо файл из конфига PHP-FPM `error_log`, либо stderr.

Когда выбран файл, то его дескриптор устанавливается функций `zlog_set_fd`.

```c
fpm_globals.error_log_fd = fd;
if (fpm_use_error_log()) {
    zlog_set_fd(fpm_globals.error_log_fd);
}
```

#### external_logger и fastcgi.logging

В секции разбора [zlog_set_external_logger](#zlog-set-external-logger) упоминалось, что `vzlog` вызывает функцию
`external_logger`.

Метод установки `zlog_external` вызывается только в child процессе.

Вызывается external_logger в самом начале `vzlog`

```c
void vzlog(const char *function, int line, int flags, const char *fmt, va_list args)
{
    char buf[MAX_BUF_LENGTH];
    size_t buf_size = MAX_BUF_LENGTH;
    size_t len = 0;
    int truncated = 0;
    int saved_errno;

    if (external_logger) {
        zlog_external(flags, buf, buf_size, fmt, args);
    }

    if ((flags & ZLOG_LEVEL_MASK) < zlog_level) {
        return;
    }
```

Следовательно, при вызове в child функции `vzlog` будет осуществлен:

* вызов `zlog_external` и лог будет отправлен в FastCGI socket (веб-серверу);
* вызов `zend_quiet_write` с отправкой лога в stderr child процессом.

Дабы логи не отправлялись в FastCGI socket в **настройках php.ini** стоит выключить опцию `fastcgi.logging`.

### Практический пример

**Пример 1**.

```php
<?php

echo "FastCGI output";

file_put_contents('php://stdout', 'To stdout');
file_put_contents('php://stderr', 'To stderr');
```

Настройки PHP (php.ini):

```ini
[PHP]
fastcgi.logging = 1
```

Выполнение кода с выключенной настройкой `catch_workers_output = no`:

* записи в `php://stdout` и `php://stderr` попадут в `/dev/null` и мы их не увидим ни в логах, ни в stderr master
* процесса;
* текст "FastCGI output" попадет в FastCGI socket и будет доставлен отправителю запроса;
* ошибки самого child процесса будут записаны с вызовом функции `zlog` с последующей записью в stderr ассоциированным
  с `/dev/null`, т.е. ошибки не будут записаны ни в логи, ни в `stderr` master процесса.

Выполнение кода с включенной настройкой `catch_workers_output = yes`:

* записи в `php://stdout` и `php://stderr` попадут в pipes и будут прочитаны master процессом;
* текст "FastCGI output" попадет в FastCGI socket и будет доставлен отправителю запроса;
* ошибки самого child процесса будут записаны с вызовом функции `zlog` с последующей записью в `stderr` ассоциированным
  с fd_stderr pipe и будут доставлены master процессу.


**Пример 2**.

PHP скрипт генерирует fatal error при попытке выполнения.

```php
<?php
function my_function(array $a) { echo $a; }
my_function(123.0);
```

Настройки PHP (php.ini):

```ini
[PHP]
log_errors = On
error_log = /tmp/php-errors.txt
fastcgi.logging = 1
```

Настройки PHP-FPM:

```ini
[global]
error_log = /proc/self/fd/2
[www]
catch_workers_output = yes
decorate_workers_output = yes
```

При выполнении этого кода (напомню, речь о запросе через веб-сервер), в `/tmp/php-errors.txt` будет записана
информация о fatal error.

```txt
[12-Jul-2022 09:56:23 UTC] PHP Fatal error:  Uncaught TypeError: Argument 1 passed to my_function() must be of the type array, float given, called in /var/www/html/public/error_handling/error_generators/fatal_error.php on line 3 and defined in /var/www/html/public/error_handling/error_generators/fatal_error.php:2
Stack trace:
#0 /var/www/html/public/error_handling/error_generators/fatal_error.php(3): my_function()
#1 {main}
  thrown in /var/www/html/public/error_handling/error_generators/fatal_error.php on line 2
```

В FastCGI socket *ничего не будет отправлено*, так как настройка php.ini `error_log` приведет к записи ошибки в файл,
а чтобы ошибка попала в `vzlog`, должен быть настроен `error_log` в php.ini на запись в stderr!

**Пример 3**.

Сгенерируем fatal error, но теперь php.ini `error_log` будет указывать на stderr.

```php
<?php
function my_function(array $a) { echo $a; }
my_function(123.0);
```

Настройки PHP (php.ini):

```ini
[PHP]
log_errors = On
error_log = /proc/self/fd/2
fastcgi.logging = 1
```

Настройки PHP-FPM:

```ini
[global]
error_log = /proc/self/fd/2
[www]
catch_workers_output = yes
decorate_workers_output = yes
```

Возникший fatal error попадет обработчиком `error_log` PHP (не путать с PHP-FPM `error_log`) запишет fatal error
в stderr child процесса (включен `catch_workers_output`). 

Также в FastCGI socket будет
[отправлено FCGI_STDERR]({{ site.baseurl }}{% link _posts/2022-05-08-php-fpm-config-structure.md %})
сообщение.

У меня веб-серверов выступает nginx, потому в логе nginx я увижу следующее

```shell
2022/07/12 10:06:34 [error] 24#24: *1 FastCGI sent in stderr: "PHP message: PHP Fatal error:  Uncaught TypeError: Argument 1 passed to my_function() must be of the type array, float given, called in /var/www/html/public/error_handling/error_generators/fatal_error.php on line 3 and defined in /var/www/html/public/error_handling/error_generators/fatal_error.php:2
Stack trace:
#0 /var/www/html/public/error_handling/error_generators/fatal_error.php(3): my_function()
#1 {main}
thrown in /var/www/html/public/error_handling/error_generators/fatal_error.php on line 2" while reading response header from upstream, client: 172.16.7.1, server: localhost, request: "GET /error_handling/error_generators/fatal_error.php HTTP/1.1", upstream: "fastcgi://192.168.65.2:9000", host: "localhost:8085"
```

Также в stderr master процесса PHP-FPM будет доставлено

```
172.16.7.1 - 12/Jul/2022:10:06:34 +0000 "GET /error_handling/error_generators/fatal_error.php" 500 /var/www/html/public/error_handling/error_generators/fatal_error.php 0.016 2097152 0.00
[12-Jul-2022 10:06:34] WARNING: [pool www] child 8 said into stderr: "NOTICE: PHP message: PHP Fatal error:  Uncaught TypeError: Argument 1 passed to my_function() must be of the type array, float given, called in /var/www/html/public/error_handling/error_generators/fatal_error.php on line 3 and defined in /var/www/html/public/error_handling/error_generators/fatal_error.php:2"
[12-Jul-2022 10:06:34] WARNING: [pool www] child 8 said into stderr: "Stack trace:"
[12-Jul-2022 10:06:34] WARNING: [pool www] child 8 said into stderr: "#0 /var/www/html/public/error_handling/error_generators/fatal_error.php(3): my_function()"
[12-Jul-2022 10:06:34] WARNING: [pool www] child 8 said into stderr: "#1 {main}"
[12-Jul-2022 10:06:34] WARNING: [pool www] child 8 said into stderr: "  thrown in /var/www/html/public/error_handling/error_generators/fatal_error.php on line 2"
```

Аналогичная ситуация и с пользовательским ошибками и trigger_error.

Отдельно отмечу, что благодаря настройке `decorate_workers_output = yes` мы видим детализацию про запись в stderr child
процессом ошибки, ну и что ошибку породил child. Стоит утсновить `decorate_workers_output = no` и у нас начнется каша
в **глобальном логе PHP-FPM**:

```
To stdout
To stderr
NOTICE: PHP message: PHP Fatal error:  Uncaught TypeError: Argument 1 passed to my_function() must be of the type array, float given, called in /var/www/html/public/error_handling/error_generators/fatal_error.php on line 3 and defined in /var/www/html/public/error_handling/error_generators/fatal_error.php:2
Stack trace:
#0 /var/www/html/public/error_handling/error_generators/fatal_error.php(3): my_function()
#1 {main}
  thrown in /var/www/html/public/error_handling/error_generators/fatal_error.php on line 2
172.16.7.1 - 12/Jul/2022:10:23:43 +0000 "GET /error_handling/default_configs.php" 200 /var/www/html/public/error_handling/default_configs.php 0.005 2097152 0.00
To stdout
To stderr
```

## Заключительное слово

Данный материал явно не простой, но я очень надеюсь, что после прочтения были прояснены многие моменты и при
необходимости настройки PHP и PHP-FPM мои посты принесут пользу. Лучше благодарностью за мой труд будут репосты,
посты в facebook, twitter, в рабочие чаты и т.п.

