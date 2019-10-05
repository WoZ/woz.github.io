---
layout: post
title: Тонкости работы с stdio в node.js
tags: [node.js]
---

Данный пост посвящен запуску сторонних приложений из node.js и дальнейшим контролем за их выполнением. Эта
задача может эффективно решаться с помощью nodejs, потому он и был выбран на одном из проектов. В
официальной документации описаны все возможные события, есть примеры использования, но для enterprise
применения не хватало информация по внутренней реализации логики работы потоков (`stdin`, `stdout`, `stderr`)
и логике их открытия и закрытия. Последней каплей стал ответ на stackoverflow с примерами в которых,
событие `exit` при остановке процесса приходит, а `close` - нет. Стоило кому-то разобраться в этой неразберихе.

Типичная задача что требуется решить выглядит просто:
* наш nodejs апликейшн запускает приложение `app`;
* контроллирует успешность запуска, а при неудачном запуске уведомляет пользоввателя или другие подсистемы;
* следит за штатным/нештатным завершением `app` и принимает решения на базе этих знаний;
* обслуживает ввод/вывод через `stdin`/`stdout`/`stderr`.

Упомянутое выше приложение `app` может вести себя нестандартно и это надо учитывать. Например, иногда приложения
продолжают работать, но закрывают поток `stdin` или `stdout`. 

Любознательные могут прочитать весь
[ответ на stackoverflow](https://stackoverflow.com/questions/37522010/difference-between-childprocess-close-exit-events),
а ниже дан более простой пример отбражающий проблему на которую я ищу ответ.

```js
// Equivalent to "yes | cat".
cp = require('child_process').spawn('yes');
cp.on('exit', console.log.bind(console, 'exited'));
cp.on('close', console.log.bind(console, 'closed'));

cpNext = require('child_process').spawn('cat');
cp.stdout.pipe(cpNext.stdin);

setTimeout(function() {
    cp.kill();
    console.log('Called kill()');
}, 500);
```

Этот пример может быть запущен на linux, unix и freebsd, я тестировал поведение на node 0.12 и выше.
Программа запускает приложение `yes` (входит почти во все дистрибутивы), которое только и делает что в цикле
пишет в `stdout` букву `y` и перевод строки. Также запускается команда `cat` которая все что поступает в `stdin`
выводит в `stdout` и ничего более не делает. `cp.stdout.pipe(cpNext.stdin)` весь вывод команды `yes` направляет
на вход команде `yes`.

Логично ожидать, что после `kill` команды `yes` произойдет закрытие `stdout` команды `yes`, а затем и закрытие
`stdin` у команды `cat`, что приведет к остановке выполнения команды `cat`.

А на деле бесконечно долго может работтать код выше, но событие `close` не придет. Вывод будет следующим:

```text
Called kill()
exited null SIGTERM
```

Добавление `cp.stdout.destroy()` после `kill()` приведет к броску события `close`, добавление `cp.stdout.destroy()`
после события `exit` приведет к броску события... Минуточку, после события `exit` программа `yes` уже остановлена,
стрим `stdout` программы `yes` не существует более. Давайте повесим слушателей на события `close` и `end` стримов
`stdout` и `stderr`. Поскольку они заявлены как `stream.Readable`, то будем руководствоваться
[официальной документацией](https://nodejs.org/docs/v8.4.0/api/stream.html#stream_class_stream_readable).

```js
// Equivalent to "yes | cat".
cp = require('child_process').spawn('yes');
cp.on('exit', console.log.bind(console, 'exited'));
cp.on('close', console.log.bind(console, 'closed'));

cp.stdout.on('close', console.log.bind(console, 'stdout closed'));
cp.stdout.on('end', console.log.bind(console, 'stdout end'));
cp.stderr.on('close', console.log.bind(console, 'stderr closed'));
cp.stderr.on('end', console.log.bind(console, 'stderr end'));

cpNext = require('child_process').spawn('cat');
cp.stdout.pipe(cpNext.stdin);

setTimeout(function() {
    cp.kill();
    console.log('Called kill()');
}, 500);
```

Результат выполнения будет следующим:

```text
Called kill()
stderr end
exited null SIGTERM
stderr closed false
```

Обратимся к официальной документации [https://nodejs.org/docs/v8.4.0/api/child_process.html#child_process_event_close](https://nodejs.org/docs/v8.4.0/api/child_process.html#child_process_event_close).

> The 'close' event is emitted when the stdio streams of a child process have been closed.
> This is distinct from the 'exit' event, since multiple processes might share the same stdio streams.

> Note that when the 'exit' event is triggered, child process stdio streams might still be open.
  
Если верить описанию, child process завершился, при завершении закрываются потоки, значит событие `close`
должно было прийти, но есть у события `exit` сноска... Обьясненение этих самых случаев в описании отсуствует.


Теперь приступим к анализу и детальному разбору работы `stdio` и первым делом обратимся к
[man по stdio](http://man7.org/linux/man-pages/man3/stdio.3.html), это касается только реализации на `C`, но большинство
языков вроде Java ипользуют схожий подход в управлении потоками, к тому же, ядро linux работает согласно этой
документации. Выделим важные моменты:

* стандартный стримы `stdin`, `stdout`, `stderr` создаются библиотекой `libc` для любой `C` программы автоматически;
* `stdin` и `stdout` являются буфризированными, а `stderr` - нет (на практике это означает, что запись в `stdout` из
программы не приводит к моментальной отправке слушателю, а накапливается и только затем отправляется);
* при выходе из `main` функции или вызове функции `exit`, стримы `stdin`, `stdout`, `stderr` закрываются и буфера
флашатся (данные отправляются слушателю);
* при аварийном завершении или вызове `abort` данные что были в буфере удаляются и не будут доставлены;
* буфер может флашится программой принудительно, а в случае с `stdout` может быть выключен.



https://www.reddit.com/r/unix/comments/6gxduc/how_is_gnu_yes_so_fast/
https://stackoverflow.com/questions/29176636/can-someone-please-explain-how-stdio-buffering-works
http://www.gnu.org/software/coreutils/manual/html_node/stdbuf-invocation.html
https://github.com/coreutils/coreutils/blob/master/src/yes.c
https://nodejs.org/api/child_process.html#child_process_event_close


Как видно, события `end` и `close` на `stdout` не произошло.

С багом можно ознакомится по ссылке
"https://stackoverflow.com/questions/37522010/difference-between-childprocess-close-exit-events"
В процессе
прототипирования и экспериментов была обнаружена заметка на stackoverflow  

Доводилось ли запускать из node.js процессы и контроллировать 

Данному посту начало положил

Существующая команда

```text
$ node test.js "ls"
[1504184499.122] stdout emits 'resume' with arguments []
[1504184499.125] stderr emits 'resume' with arguments []
[1504184499.125] stdin  emits 'resume' with arguments []
[1504184499.126] stdin  emits 'readable' with arguments []
[1504184499.126] stdin  emits '_socketEnd' with arguments []
[1504184499.127] stdin  emits 'prefinish' with arguments []
[1504184499.127] stdin  emits 'finish' with arguments []
[1504184499.127] stdin  emits 'end' with arguments []
[1504184499.128] stdout emits 'data' with arguments "test.js
"
[1504184499.128] stderr emits 'readable' with arguments []
[1504184499.129] stderr emits '_socketEnd' with arguments []
[1504184499.129] stderr emits 'end' with arguments []
[1504184499.129] stderr emits 'prefinish' with arguments []
[1504184499.129] stderr emits 'finish' with arguments []
[1504184499.129] spawn  emits 'exit' with arguments [0,null]
[1504184499.130] stderr emits 'close' with arguments [false]
[1504184499.130] stdin  emits 'close' with arguments [false]
[1504184499.130] stdout emits 'readable' with arguments []
[1504184499.130] stdout emits '_socketEnd' with arguments []
[1504184499.130] stdout emits 'end' with arguments []
[1504184499.130] stdout emits 'prefinish' with arguments []
[1504184499.130] stdout emits 'finish' with arguments []
[1504184499.130] stdout emits 'close' with arguments [false]
[1504184499.130] spawn  emits 'close' with arguments [0,null]
```

Несуществующая команда

```text
$ node test.js "lsss"
[1504184596.861] spawn emits 'error' with arguments [{"code":"ENOENT","errno":"ENOENT","syscall":"spawn lsss","path":"lsss","spawnargs":[]}]
[1504184596.864] stdout emits 'resume' with arguments []
[1504184596.864] stderr emits 'resume' with arguments []
[1504184596.864] stdin emits 'resume' with arguments []
[1504184596.865] stdout emits 'readable' with arguments []
[1504184596.866] stdout emits '_socketEnd' with arguments []
[1504184596.866] stdout emits 'end' with arguments []
[1504184596.867] stdout emits 'prefinish' with arguments []
[1504184596.867] stdout emits 'finish' with arguments []
[1504184596.867] stderr emits 'readable' with arguments []
[1504184596.868] stderr emits '_socketEnd' with arguments []
[1504184596.868] stderr emits 'end' with arguments []
[1504184596.868] stderr emits 'prefinish' with arguments []
[1504184596.868] stderr emits 'finish' with arguments []
[1504184596.868] stderr emits 'close' with arguments [false]
[1504184596.868] stdout emits 'close' with arguments [false]
[1504184596.868] spawn emits 'close' with arguments [-2,null]
[1504184596.868] stdin emits 'close' with arguments [false]
```

Команда что убивается по SIGKILL

```text
$ node test.js node selfKiller.js
[1504185101.006] stdout emits 'resume' with arguments []
[1504185101.010] stderr emits 'resume' with arguments []
[1504185101.010] stdin emits 'resume' with arguments []
[1504185101.079] stdout emits 'data' with arguments "going to kill myself!
"
[1504185101.090] stderr emits 'readable' with arguments []
[1504185101.091] stderr emits '_socketEnd' with arguments []
[1504185101.092] stderr emits 'end' with arguments []
[1504185101.093] stderr emits 'prefinish' with arguments []
[1504185101.093] stderr emits 'finish' with arguments []
[1504185101.093] stderr emits 'close' with arguments [false]
[1504185101.093] stdout emits 'readable' with arguments []
[1504185101.094] stdout emits '_socketEnd' with arguments []
[1504185101.094] stdout emits 'end' with arguments []
[1504185101.095] stdout emits 'prefinish' with arguments []
[1504185101.095] stdout emits 'finish' with arguments []
[1504185101.095] stdin emits 'readable' with arguments []
[1504185101.095] stdin emits '_socketEnd' with arguments []
[1504185101.095] stdin emits 'prefinish' with arguments []
[1504185101.095] stdin emits 'finish' with arguments []
[1504185101.096] stdin emits 'end' with arguments []
[1504185101.097] spawn emits 'exit' with arguments [null,"SIGKILL"]
[1504185101.097] stdin emits 'close' with arguments [false]
[1504185101.097] stdout emits 'close' with arguments [false]
[1504185101.097] spawn emits 'close' with arguments [null,"SIGKILL"]
```

Команда что закрывает stdout, stderr и stdin

```text

```