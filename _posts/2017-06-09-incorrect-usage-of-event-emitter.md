---
layout: post
title: Неверное использование EventEmitter
tags: [node.js, EventEmitter, consul, memory leak]
---

Сегодня хочу затронуть тему некорректного использования EventEmitter и ошибок приводящих к утечкам памяти.

При разработке сервисов на Node.js часто приходится внутри сервисов создавать другие объекты-сервисы, реализующие
EventEmitter, подписываться на события которые может отправить объект-сервис и реагировать на них. Примеров
использования можно привести множество:

* [redis клиент](https://github.com/NodeRedis/node_redis#connection-and-other-events) может генерировать уведомления о
реконекте `reconnecting`, о закрытии соединения `end`;
* [memcached клиент](https://github.com/3rd-Eden/memcached#events) генерирует события `failure` и `reconnecting` и т.д.;
* [метод watch consul клиента](https://github.com/silas/node-consul#consulwatchoptions) эмитит события при изменениях.

На днях, проводя code review, я нашел ошибку приводящую к утечке памяти. Причиной ошибки стало некорректное
использование `consul.watch`. Это побудило написать этот пост и детально разобрать возможные проблемы при
использовании EventEmitter и сделать ряд рекомендаций для начинающих разработчиков.

Кратко поясню суть проблемы. Разрабатывался сервис, задача которого состояла в слежении за health статусом серверов.
*consul* поддерживает [blocking queries](https://www.consul.io/api/index.html#blocking-queries).
Эта замечательная фича позволяет отправить
[long polling](https://stackoverflow.com/questions/11077857/what-are-long-polling-websockets-server-sent-events-sse-and-comet)
запрос и получить уведомление сразу как произойдет изменение состояния сервисов. Используя ее не нужно опрашивать
статус серверов с определенным интервалом, можно сделать запрос и ждать когда он выполнится. Как уже упоминалось выше,
этот сервис слежения за health статусом серверов подписывается на события `change` и `error`. А поскольку сервис
сам по себе может находиться в состоянии active и inactive, то когда его переводят в состояние inactive, он должен
перестать следить за изменением списка серверов и активизироваться только после повторного запроса слежения. Во время
переключения состояния сервиса и была допущена ошибка разработчиком.

Чтобы не погружаться в освоение специфики работы с *consul* и его клиентом, будем работать с примерами повторяющими
его поведение. Все исходные коды что приводятся в посте
[доступны на github](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter). Что же, создадим класс
[`InternalService`](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter/blob/master/InternalService.js):

```js
class InternalService extends EventEmitter {
  constructor(dataLength, disableLogging = false) {
    super();
    this._disableLogging = disableLogging;
    this._data = '-'.repeat(dataLength);
  }

  start() {
    this._intervalId = setInterval(() => {
      if (!this._disableLogging) {
        log(util.format('emits data to %s listeners', this.listenerCount('data')));
      }
      this.emit('data', this._data);
    }, 100);
  }

  stop() {
    if (this._intervalId) {
      clearInterval(this._intervalId);
    }
  }
}
```

Объекты данного класса после запуска метода `InternalService.start` будут генерировать раз в 100мс событие пока
не будет явно выполнен метод `InternalService.stop`. Это эмуляция поведения в точности как у `consul.watch`.

Теперь создадим несколько вариантов сервисов использующих `InternalService` с ошибочной логикой.


### Множественная подписка на одно и то же событие


Такую ошибку совершают начинающие разработчики или бывалые по невнимательности

```js
class ExternalService {
  constructor() {
    this.internalEmitter = new InternalService(10000000);
  }

  start() {
    this.internalEmitter.on('data', this._onDataEvent);
    this.internalEmitter.start();
  }

  stop() {
    this.internalEmitter.stop();
  }

  _onDataEvent(data) {
    log('new data from InternalService');
  }
}
```

Полный исходник на github -
[`duplication_of_listeners.js`](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter/blob/master/duplication_of_listeners.js).

Вроде все выглядит корректно: создается сервис в конструкторе и он не меняется во время жизни объекта класса
`ExternalService`, `InternalService.start` и `InternalService.stop` вызываются корректно, функция внутри `setInterval`
будет чистится [Garbage Collector'ом](https://blog.risingstack.com/node-js-at-scale-node-js-garbage-collection/).
Но проблема кроется в навешивании множества хэндлеров `_onDataEvent` на одно и то же событие.

Разберем детально проблему. Для проверки поведения я написал следующий код:

```js
const extService = new ExternalService();

setInterval(() => {
  extService.start();
  setTimeout(() => {
    extService.stop();
  }, 150);
}, 200);

```

Данный код с интервалом в 200мс стартует слежение за InternalService и останавливает его спустя 150мс после запуска.
Таким образом, чуть больше чем за 1 секунду будет произведено 5 запусков и остановок сервиса.

Вывод будет примерно следующим:

```text
[1497020906.645] emits data to 1 listeners
[1497020906.668] new data from InternalService
[1497020906.850] emits data to 2 listeners
[1497020906.851] new data from InternalService
[1497020906.851] new data from InternalService
[1497020907.051] emits data to 3 listeners
[1497020907.051] new data from InternalService
[1497020907.051] new data from InternalService
[1497020907.051] new data from InternalService
[1497020907.255] emits data to 4 listeners
[1497020907.255] new data from InternalService
[1497020907.255] new data from InternalService
[1497020907.255] new data from InternalService
[1497020907.255] new data from InternalService
```

Документация Node.js [явно говорит об этой проблеме](https://nodejs.org/api/events.html#events_emitter_on_eventname_listener),
но все часто об этом забывают. Исправить ошибку можно изменив код класса следующим образом:

```js
class ExternalService {
  // ...
  stop() {
    this.internalEmitter.removeListener('data', this._onDataEvent);
    this.internalEmitter.stop();
  }
  // ...
}
```

Исправленная версия доступна по адресу
[`fixed_duplication_of_listeners.js`](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter/blob/master/fixed_duplication_of_listeners.js).

Запустим и проверим что ошибка не повторяется:

```text
[1497024994.577] emits data to 1 listeners
[1497024994.607] new data from InternalService
[1497024994.781] emits data to 1 listeners
[1497024994.782] new data from InternalService
[1497024994.983] emits data to 1 listeners
[1497024994.983] new data from InternalService
[1497024995.187] emits data to 1 listeners
[1497024995.187] new data from InternalService
[1497024995.399] emits data to 1 listeners
[1497024995.399] new data from InternalService
[1497024995.604] emits data to 1 listeners
[1497024995.604] new data from InternalService
```

Стоит отметить важный нюанс, `removeListener` производит сравнение по ссылке. Об этом свидетельствует код
[events.js](https://github.com/nodejs/node/blob/afc59a9f50b8a48c405ce35452111309d6913f47/lib/events.js#L355), потому
для корректной работы нужно передавать методу точно тот же метод/функцию что была передана аргументом метода
`EventEmitter.on()`.

[Код incorrect_listener_remove.js](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter/blob/master/incorrect_listener_remove.js)
будет работать некорректно и память будет течь, одни и те же события будут доставляться множеству хэндлеров:

```js
class ExternalService {
  // ...
  start() {
    this.internalEmitter.on('data', this._onDataEvent.bind(this));
    this.internalEmitter.start();
  }

  stop() {
    this.internalEmitter.removeListener('data', this._onDataEvent.bind(this));
    this.internalEmitter.stop();
  }
  // ...
}

```


### Неочищенные таймеры


В реальных приложениях данная ошибка встречается чаще, разработчики чистят и чистят корректно функции-слушатели,
переприсваивается ссылка на объект, что должно привести к очистке объекта. Но этого не происходит. Причиной
того является наличие в `InternalService` таймера. Покуда таймер остается активен GC не произведет очистку памяти,
ссылка на объект потеряна в приложении, но он продолжает существовать. Приведу пример... За основу будет взят все тот
же `InternalService` класс, изменен будет класс `ExternalService`, исходный код в 
[примере unstopped_timers.js](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter/blob/master/unstopped_timers.js):

```js
class ExternalService {
  constructor() {
  }

  start() {
    this.internalEmitter = new InternalService(10000000);
    this.internalEmitter.on('data', this._onDataEvent);
    this.internalEmitter.start();
  }

  stop() {
    this.internalEmitter.removeListener('data', this._onDataEvent);
  }

  _onDataEvent(data) {
    log('new data from InternalService');
  }
}
```

Запуск теста будет осуществляться тем же кодом что и в предыдущем примере:

```js
const extService = new ExternalService();

setInterval(() => {
  extService.start();
  setTimeout(() => {
    extService.stop();
  }, 150);
}, 200);
```

Вывод будет следующим

```text
[1497260785.333] emits data to 1 listeners
[1497260785.359] new data from InternalService
[1497260785.493] emits data to 0 listeners
[1497260785.537] emits data to 1 listeners
[1497260785.537] new data from InternalService
[1497260785.596] emits data to 0 listeners
[1497260785.638] emits data to 0 listeners
[1497260785.700] emits data to 0 listeners
[1497260785.740] emits data to 1 listeners
[1497260785.740] new data from InternalService
[1497260785.740] emits data to 0 listeners
[1497260785.801] emits data to 0 listeners
[1497260785.841] emits data to 0 listeners
[1497260785.841] emits data to 0 listeners
[1497260785.906] emits data to 0 listeners
[1497260785.942] emits data to 1 listeners
[1497260785.942] new data from InternalService
```

Детально разберем происходящее. Сервис стартует, `InternalService` отправил событие одному подписчику, подписчик
получил событие, сервис остановился, а затем мы видим `[1497260785.493] emits data to 0 listeners`. Поскольку
в методе `ExternalService.stop` присутствует `removeListener`, то у события **data** нет больше слушателей, но
таймер активен и события пытаются отправляться в никуда. Стоит обратить внимание, что
`new data from InternalService` эмитится один раз между start-stop и не происходит многократной доставляемости
событий на хэндлер как в первом примере.

Устранить эту ошибку можно очищая функцией `clearInterval` таймер в `InternalService`, к слову у класса уже есть
метод `stop` и он делает именно это.
[Исправленный код fixed_unstopped_timers.js](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter/blob/master/fixed_unstopped_timers.js)
приведен ниже:

```js
class ExternalService {
  // ...
  stop() {
    this.internalEmitter.removeListener('data', this._onDataEvent);
    this.internalEmitter.stop();
  }
  // ...
}
```

Вывод будет следующим:

```text
[1497099930.422] emits data to 1 listeners
[1497099930.424] new data from InternalService
[1497099930.670] emits data to 1 listeners
[1497099930.671] new data from InternalService
[1497099930.869] emits data to 1 listeners
[1497099930.869] new data from InternalService
[1497099931.074] emits data to 1 listeners
[1497099931.074] new data from InternalService
```

Подобные ошибки могут происходить не только с активными таймерами, но и с запущенным в `InternalService`,
web сервером, события слежения за изменениями файловой системы, etc, в общем, всем тем что в `libuv` добавляет колбэки.


### Нужно ли удалять хэндлеры?


Во втором примере присутствовал явный вызов `removeListener`. Всегда ли нужно отписываться от событий перед
удалением ссылки на объект?

Код из предыдущих примеров не может дать ответ на этот вопрос. Дело в том, что мы остановили таймер,
некому генерировать события, но остается риск того, что где-то осталась ссылка на объект `InternalService` и
он не будет очищен GC. С большой долей вероятности сказать течет ли память можно добавив таймер с выводом
использования памяти. Я специально оговорился про "большую долю вероятности", потому что для однозначного ответа
потребуется анализировать heap с течением времени. Этому вопросу будет посвящен отдельный пост. А пока
проведем эксперимент и удалим из кода второго примера строчку с `removeListener`
(код доступен по адресу
[remove_or_not_remove_handlers.js](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter/blob/master/remove_or_not_remove_handlers.js)
):

```js
class ExternalService {
  constructor() {
    this._bytesReceived = 0;
  }

  getBytesReceived() {
    return this._bytesReceived;
  }

  start() {
    this.internalEmitter = new InternalService(10000000, true);
    this.internalEmitter.on('data', this._onDataEvent.bind(this));
    this.internalEmitter.start();
  }

  stop() {
    this.internalEmitter.stop();
  }

  _onDataEvent(data) {
    this._bytesReceived += data.length;
  }
}


const extService = new ExternalService();

setInterval(() => {
  log(util.format('heapUsed = %d, bytes received = %d', process.memoryUsage().heapUsed, extService.getBytesReceived()));
}, 1000);

setInterval(() => {
  extService.start();
  setTimeout(() => {
    extService.stop();
  }, 150);
}, 200);
```

Запустим код и увидим примерно такой вывод:

```text
[1497259316.038] heapUsed = 3971112, bytes received = 40000000
[1497259317.079] heapUsed = 4119592, bytes received = 90000000
[1497259318.089] heapUsed = 4413432, bytes received = 140000000
// ...
// происходит рост
// ...
[1497259388.315] heapUsed = 6051464, bytes received = 3600000000
[1497259389.316] heapUsed = 6083672, bytes received = 3650000000
[1497259390.321] heapUsed = 4677640, bytes received = 3690000000
[1497259391.323] heapUsed = 4696800, bytes received = 3740000000
// ...
// и снижение обьяма потребляемой памяти, видно что отработал GC
// ...
[1497259439.482] heapUsed = 5652952, bytes received = 6120000000
[1497259440.486] heapUsed = 5672112, bytes received = 6170000000
[1497259441.486] heapUsed = 5691272, bytes received = 6220000000
[1497259442.487] heapUsed = 4729712, bytes received = 6260000000
[1497259443.488] heapUsed = 4749312, bytes received = 6310000000
// ...
// и снова отработал GC
// ...
```

Память освобождается.


Чтобы понимать поведение сборщика мусора следует понимать как работает EventEmitter, потому предлагаю [ознакомиться
с исходным кодом](https://github.com/nodejs/node/blob/afc59a9f50b8a48c405ce35452111309d6913f47/lib/events.js).
Любой объект что наследует EventEmitter
[приобретает поле `_events` являющееся объектом при вызове конструктора](https://github.com/nodejs/node/blob/afc59a9f50b8a48c405ce35452111309d6913f47/lib/events.js#L72).
Метод `EventEmitter.on` (алиас [метода `EventEmitter.addListener`](https://github.com/nodejs/node/blob/afc59a9f50b8a48c405ce35452111309d6913f47/lib/events.js#L298))
[добавляет в объект `_events`](https://github.com/nodejs/node/blob/afc59a9f50b8a48c405ce35452111309d6913f47/lib/events.js#L272)
по ключу с именем события переданные функции-обработчики. Метод `EventEmitter.emit`
[вызывает эти обработчики](https://github.com/nodejs/node/blob/afc59a9f50b8a48c405ce35452111309d6913f47/lib/events.js#L155).

Таким образом, при вызове `this.internalEmitter.on('data', this._onDataEvent);` ссылка на функцию `this._onDataEvent`
сохраняется внутри поля самого же объекта `this.internalEmitter`. А когда теряется ссылка на объект
`this.internalEmitter`, то GC удаляет и все ссылки в свойствах объекта, а это равносильно вызову `removeListener`,
потому что последний делает то же самое, но явно.

Потому в ситуации описанной выше ничего страшного не произойдет если явно не вызывать `removeListener`.

### Резюме

Из всего вышесказанного отмечу главные пункты:

* Следите за условиями при которых происходит подписка на события и не допускайте множественной подписки в местах где
это не нужно. [patched_event_emitter.js](https://github.com/WoZ/examples-incorrect-usage-of-eventemitter/blob/master/patched_event_emitter.js)
содержит пример того как можно переопределить методы EventEmitter для слежения и отладки;
* Следите за таймерами, открытыми и прослушиваемыми портами, событиями файловой системой. Отсутствие остановок и отписок
приведет к утечками памяти и не прогнозируемому поведению;
* Думайте при написании кода о том как GC будет отрабатывать.

**UPD.** Рекомендую ознакомиться с еще одной моей статьей посвященной [паттерну `Event Emitter` и объясненим почему
не стоит реализовывать двунаправленное общение]({{ site.baseurl }}{% link _posts/2017-08-02-bidirectional-event-emitting.md %}).