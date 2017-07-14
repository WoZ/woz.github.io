---
layout: post
title: let, const и Temporal Dead Zone 
tags: [node.js, TDZ, ES6, javascript, temporal dead zone]
---

На каждом собеседовании javascript разработчиков я или мой напарник задаем вопрос об
отличии `let`, `const` и `var` в ES6. Благо базовые моменты люди знают и понимают, говорят о
блочной зоне видимости, возможности переопределения переменных обьявленных с `let` и невозможностью
переопределить объявленные как `const`. `var` сравнивается с `let` в контексте отсутствия
[поднятия переменных](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting).

Но как будет работать код ниже?

```js
const util   = require('util');
const EventEmitter = require('events').EventEmitter;

class CrazyEmitter extends EventEmitter {
    constructor() {
        super();
        setInterval(() => {
            this.emit(Math.random() > 0.5 ? 'resolve' : 'reject');
        }, 1000);
    }
}

class Watcher {
    firstEvent(emitter) {
        return new Promise((resolve, reject) => {
            const onResolve = () => {
                console.log('resolve');
                emitter.removeListener('reject', onReject);
                resolve();
            };

            const onReject = () => {
                console.log('reject');
                emitter.removeListener('resolve', onResolve);
                reject();
            };

            emitter.once('resolve', onResolve);
            emitter.once('reject', onReject);
        });
    }
}

const watcher = new Watcher();
const emitter = new CrazyEmitter();

function runNext() {
    watcher.firstEvent(emitter).then(runNext).catch(runNext);
}

runNext();
````

На этом вопросе большинство опрошенных сыпались, предполагали бросок `ReferenceError: onReject is not defined`. 
Людей можно понять, ведь множество ресурсов дают неполную или некорректную информацию:

* [learn.javascript.ru](https://learn.javascript.ru/let-const) пишет:

> Переменная let видна только после объявления.
> Как мы помним, переменные var существуют и до объявления. Они равны undefined.
> С переменными let всё проще. До объявления их вообще нет.

* [developer.mozilla.org](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Statements/let)
дает больше информации:

> В стандарте ECMAScript 2015 переменные, объявленные оператором let, переносятся в начало блока.
> Но если вы сошлетесь в блоке на переменную, до того как она объявлена оператором let, то это приведет
> к выбросу исключения ReferenceError, потому что переменная находится во "временной мертвой зоне" с
> начала блока и до места ее объявления. (В отличии от переменной, объявленной через var, которая просто
> будет содержать значение undefined)

* На [strongloop.com](https://strongloop.com/strongblog/es6-variable-declarations/) дают примеры ошибок 
и решают проблему путем переноса инициализации переменной в самое начало функции.
Почти все упоминания и примеры сводятся к тому, что надо объявлять переменную выше по коду до ее использования.

Так как же будет работать код?
Он будет работать корректно. Большинство ресурсов упускают важную деталь при определении Temporal Dead Zone.
Переменная все так же попадает в замыкания даже до момента ее инициализиции, ее видят функции. `ReferenceError`
бросается когда имеет место попытка обращения к переменной раньше ее инициализации, но речь именно об инструкциях
обращения и их порядке, а не о расположении обьявления переменнойы в коде.

Если говорить об инструкциях на примере выше, то движок будет обрабатывать код следующим образом (на пальцах):

1. сперва строит LexicalEnvironment функции в promise;
2. на этапе построения создаются переменные объявленные с использованием ключевых слов `let` и `const`, но не
  происходит инцииализации переменных;
3. начинается выполнение инструкций;
4. если инструкция пытается обратиться к переменной объявленной с `let` или `const`, но которая еще неинициализирована,
  то произойдет бросок `ReferenceError`;
5. при встрече инструкции по созданию функции `onResolve` создастся LexicalEnvironment ассоциированный с этой функцией
  и привязывается LexicalEnvironment из первого пункта, другими словами, создается замыкание в которые попали
  переменные `onReject`, `emitter`, `resolve`, `reject`, etc
6. обращения к `onReject` не происходило, потому никаких исключений нет
7. к моменту когда начнет выполняться `onResolve` переменная `onReject` уже будет инициирована и обращение не вызовет
  исключения

Более строгое описание в
[спецификации ECMAScript 2015](http://www.ecma-international.org/ecma-262/6.0/#sec-let-and-const-declarations).

Пишите код правильно и читайте правильные мануалы. До встречи!
  
  