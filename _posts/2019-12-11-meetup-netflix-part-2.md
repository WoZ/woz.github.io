---
layout: post
title: Meetup. Об IT без галстуков. Встреча с инженером Netflix, беседа о ДНК компании и о секретах ее успеха. Часть 2
image: /img/previews/meetup_netflix_p2.jpg
tags: [netflix, culture, meetup, recruitment, freedom, responsibility, hr, salary, life in usa]
---

Сегодня публикую вторую часть оцифровки нашего миитапа с Арсеном Костенко из Netflix. Напомню, что сама встерча 
прошла 10 ноября и собралось 120 человек.

Тем кто не читал [первую часть]({{ site.baseurl }}{% link _posts/2019-11-27-meetup-netflix-part-1.md %}) рекомендую
начать с нее. В первой части шла речь собеседованиях, найме, доходе, акциях, опционах и затронули культуру компании,
которая умещается в двух словах: freedom & responsobility.


В этой части речь пойдет о:

* культуре в Netflix;
* имеет ли инженер понимание зачем делается система? насколько глубоко?;
* вкладе менеджера и разработчиков в общее дело;
* метриках, SLA, agile и self management;
* об организации команд, межкомандном взаимодействии и коммуникацияхж
* PDP (personal development plan), оценки 360;
* снова о пересмотре дохода, хотя это было в части 1;
* DevOps и деплоймент в проектах.

В конце этой статьи ссылки на наши контактные данные. Те кто уверен в своих силах могут запросить reference check у
Арсена.

> Оцифрованная версия **для чтения на русском языке**, но те кто понимают **украинский** могут **посмотреть видео**.
2 часа 17 минут, правда, но на прочтение текста ниже потребтуеся 10-15 минут.

> <iframe width="560" height="315" src="https://www.youtube.com/embed/1so_s4y6b54" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Поехали! **ДМ** - это я, **АК** - это Арсен Костенко. **Зал** - это зал)

**ДМ**: Итак, давай перейдём к вопросам о культуре и ценностях Netflix. Потому что компания делает большие ставки именно на культуру. Даже перед тем, как заполнять какой-нибудь job profile (на job портале самого Netflix, приметка), они просят посмотреть видео об их культуре, и понять, соотносятся ли твои принципы с их принципами. Поэтому давай начнём вообще с онбординга и адаптации, когда ты пришёл в компанию, что было в первую неделю, в первый месяц? Как тебя интегрировали в эту культуру? Что вообще такое культура Netflix?

**АК**: Как меня интегрировали в культуру Netflix? Знаете, такого, как студенческое посвящение, когда ты надеваешь какой-то балахон, тебе закрывают глаза и просят что-то выпить, конечно, не было. 

**ДМ**: А в Украине такое есть (смех зала)

**АК**: Ну, было бы интересно снять об этом фильм, я не знаю...

**ДМ**: А покажете у себя?

**АК**: Это уже к контенту. Я сейчас пытаюсь вспомнить, что было такого особенного...есть определённые ресурсы, которыми ты можешь пользоваться, если ты не понимаешь культуры. Очень много общения и того, что называется over communication, когда лучше по 50 раз повторить одно и то же просто, чтобы была 100% уверенность в том, что человек услышал то, что вы хотите сказать. Вместо ожиданий, что человек всё понимает априори. Есть ли какой-то механизм, который в это погружает? Мне кажется, наоборот, у Google или Facebook этого механизма больше, потому что у них есть период, когда вы что-то делаете, например, за первую неделю сделать свой первый commit, который пойдёт в продакшн, и всякое такое. За счёт того, что у Netflix есть freedom & responsibility, каждая команда сама решает, как она это будет делать. Каждый менеджер по-своему решает. И также это означает, за счёт культурной части собеседования, что человек, который приходит в команду, уже а) имеет определённые ожидания, б) понимает, что он сам должен быть проактивным. То есть, никто вместо меня эту культуру не привьёт. В том числе, я сам должен узнать, что она означает, должен её демонстрировать и проявлять. То есть, это больше о проактивности, чем о восприятии. Это первая часть. Вторая часть - как это выглядит в реальном мире? В реальном мире это выглядит примерно так - мне говорят: “Вот твоё место, вот твой компьютер. Чем тебе помочь?”

**ДМ**: То есть, ты сам в рамках своего департамента можешь выбирать,чем заниматься и анализируешь метрики, чтобы понять, что работает не так?

**АК**: Мы с тобой перешли от культуры до технических заданий. Это то направление, в котором ты хочешь обсуждать или ты хочешь вернуться к культуре?

**ДМ**: Вообще это коррелирует с культурой. Но я хочу понять, где у тебя появляется responsibility?

**АК**: Я не знаю, я не уверен, что я тебя полностью понял. Смотри, когда человека нанимают в компанию, в основном уже есть идея того, чем заниматься и какую работу делать. 

**ДМ**: Это идея, ты говоришь об идее. Но кто-то из людей детализирует, что, например, таким то образом мы хотим сделать такую то вещь? Или тебе говорят, что есть общие пункты, над которыми мы будем работать в этом году, и ты уже сам делаешь выбор, что может иметь наибольшее значение?

**АК**: Я попробую объяснить то, что было со мной. Была идея создания определённой системы, микросервиса, мы его условно называем MNDB. Я пришёл для того, чтобы усилить команду, которая будет его создавать. Было ли детально расписано техническое задание для меня? Нет. Были ли детально расписаны требования к сервису на каком-то базовом уровне? Да. То есть, требования были, технических решений не было. Моё дело - добавить технические решения к требованиям так, чтобы всё сработало. Не моё дело - говорить, почему именно эта система должна существовать. Но, если я считаю, что система могла бы быть в чём-то лучше или где-то есть недостатки, я могу пойти и узнать больше. Не факт, что это изменит решение о построении этой системы, но это может значительно влиять на её архитектуру и на поле задач, которые она решает. 

**ДМ**: А менеджеры рассказывают, в чём вообще мотивация создавать эту систему? Если ты придёшь и спросишь, тебе дадут ответ?

**АК**: Да. Да, и довольно расширено. Объяснение и описание того, почему необходимо это строить, есть, они задокументированы, и их часто проговаривают не только менеджеры, но и сами участники команды между собой. 

**ДМ**: Проводятся ли какие-то чеки в процессе создания, чтобы понять, куда вы движетесь и в правильном ли направлении?

**АК**: Да, чеки есть, процессы есть. То есть, существует нормальный этап внедрения системы в продакшн, когда вы её внедрили, вы мониторите, насколько её используют, кто использует, как использует, и потом постоянно на короткой ноге с вашими непосредственными пользователями - почему они её используют, какие были проблемы, что было бы лучше, и так вы развиваетесь всё дальше и дальше.

![Meetup Photo 5]({{ "/img/meetup_netflix/photo_5.jpg" | absolute_url }}){: .in-post-img-center .in-post-img-border}

**ДМ**: Хорошо. Тогда вопрос о менеджменте. Как вообще происходит коммуникация с менеджером, какая его роль в твоей работе, как вы коммуницируете?

**АК**: Это зависит от нас самих - как налажена коммуникация, есть ли взаимопонимание и доверие. Если у вас нет доверия, то коммуникация происходит чаще, чтобы это доверие и взаимопонимание построить. Это основная цель. В моём случае это было от 1 раза в неделю до 1 раза в две недели, примерно по 30-60 минут. 

**ДМ**: Ок, а насчёт vision, который формулировали менеджеры - этапе формирования идей. Это делали менеджеры или кто?

**АК**: Это общее решение. В нашем случае это было примерно так - я и моя команда строили похожие системы несколько раз. В какой-то момент стало понятно, что мы можем построить похожую систему в пятый раз, но лучше мы построим платформу для построения этой системы. 

**ДМ**: То есть, решение было командным? Кому пришла такая идея, что нужна какая-то платформа?

**АК**: Какая разница, кому пришла эта идея? 

**ДМ**: Я пытаюсь понять, насколько техническая команда, менеджеры делают вклад в работу компании. Кто вообще является генератором этих идей, куда двигаться? Это же инновационная составляющая.

**АК**: Это инновационная составляющая, но я не уверен, что ты можешь её формализовать, а потом  воспроизвести. Это зависит от разных команд. Есть команды, в которых программисты, так называемые IC (individual contributors), фонтанируют идеями, и роль менеджера заключается в том, чтобы соглашаться. Это один паттерн работы. Другой паттерн работы это когда наоборот есть отдельные люди, которые занимаются продуктом, и говорят: “Нам нужно это или то”. В нашем случае это синергия всего. У нас есть идеи, они много обсуждаются внутри, а наверх их подают только тогда, когда получают внешнюю валидацию от менеджеров и продакт-менеджеров. Когда есть валидация, и мы понимаем, что это реальный бизнес кейс, мы начинаем более активно включаться. Но я не уверен, что это уникальный рецепт, который сработает во всех компаниях.

**ДМ**: Ок. Вы используете KPI или какие-то метрики? Как вы на них смотрите?

**АК**: Это сложный вопрос. У меня есть метрики для себя, на которые я смотрю. Я делаю их в джире для себя. Любая часть кода, которую я коммичу, должна быть привязана к той или иной джире, просто для того, чтобы я мог посмотреть, сколько я на неё потратил времени, сколько времени прошло между тем, как я её создал и начал делать, а потом тем, как я начал делать и закончил. Я не смотрю на эти отчёты постоянно, но раз в квартал или год, я смотрю, на что больше всего я потратил времени, в каком направлении было больше работы, и в каком направлении я ожидал больше работы. Это индивидуально.

**ДМ**: Есть ли какие-то цели у команды и как измеряется вообще перформанс самой команды?

**АК**: Мне сложно ответить. Я не знаю...мне стыдно, я должен был бы знать, но...

**ДМ**: То есть, я могу сделать предположение, что у вас, как у команды, нет привязки к каким-то KPI? Делать столько то задач, внедрять столько то инноваций?

**АК**: Смотри, это может сейчас прозвучать слишком непрофессионально, но если команда может привязать непосредственно свою работу к общекорпоративным метрикам (есть метрики, которые публикуются раз в квартал), и показать, что её работа влияет на эти метрики, то это очень круто. В нашем случае мы можем привязать свою работу к тем метрикам, которые являются производными от тех метрик, которые мы публикуем.

**ДМ**: Ок, это если говорить про измерение деньгами. А существуют ли какие-нибудь нематериальные оценки? Когда вы разрабатываете сервис, у вас наверняка есть технические метрики перформанса этой системы?

**АК**: Да, но мы уже говорим о SLA, это уже не продуктивность самого программиста.

**ДМ**: Если ты говоришь про SLA, то его можно вычислить у целой команды. Например, необходимо, чтобы держался определённый SLA. Некоторые компании делают именно так, когда привязывают KPI к какому-то отделу. Если SLA там полностью разрушен, то команда перформит плохо. И ты смотришь по этим характеристикам, понимаешь, где у тебя есть проблемы, над чем необходимо поработать. 

**АК**: Да, такое есть. Но это легко привязать к тем командам, которые ближе всего к пользователю. Где SLA непосредственно влияет на бизнес метрики. В случае, если у тебя есть сервис, который не прямо зависит от бизнес метрик, ты в какой-то момент наткнёшься на вопрос “А как мы транслируем одно в другое?” Потому что в результате цель компании, в Netflix, это развлекать всех. Но, если мы не можем чётко сказать, что именно мы делаем для этого, то всё остальное является просто способом навязать какие-то метрики без чёткого понимания, что мы делаем. Я достаточно понятно объяснил?

**ДМ**: Да, да. Это процессы, которые у вас существуют, как чек-лист того, что у вас есть. Кстати, вы используете какие-то agile?

**АК**: Мы встречаемся с командой, мы не называем это “скрам”, мы не называем это “канбан”, но мы встречаемся два раза в неделю и много-много раз в день. 

**ДМ**: У вас кто-то знает, что такое agile manifesto, все эти вещи?

**АК**: Да, он у нас не распечатан и не прибит на стену, но, когда ты говоришь “convention over configuration”, люди понимают, о чём ты говоришь.

**ДМ**: А реально это помогает в работе или нет? Например, вы можете использовать какие-то гетерогенные практики, не называя это agile, scrum или kanban?

**АК**: В моей команде мы как раз используем гетерогенные практики. У нас нет религиозного следования ни одной задекларированной методологии.

**ДМ**: И команды могут сами решать?

**АК**: Да. Есть команды, которые очень религиозно относятся к канбану, и ты заходишь, а у них там всё, как на ладони, видно.  

**ДМ**: Ок. А что касается daily management? От постановки задач. контроля времени, исполнения задач? Менеджер или кто-то другой вообще следит за количеством velocity команды, за количеством закрытых задач за какой-то отрезок времени или привязок нет?

**АК**: За velocity следят, но на уровне бизнес-задач. Не на уровне тикетов или каких-то конкретных открытых задач в джире. В джире я это делаю сам для себя. 

**ДМ**: Часто в джире отслеживаешь?

**АК**: Лично я да, мне это очень помогает. 

**ДМ**: Ок. То есть, если вы работаете от идей, то у вас фиксация происходит от идей или какое-то декомпозирование на техническое задание от какой-то роли, например, технического лида или тимлида, а потом уже разработчики работают? Или каждый от идеи работает и самоорганизовывается?

**АК**: Ну, нас не много. У нас команда, что называется, pizza-sized, то есть, команда, которую ты можешь накормить одной пиццей - от 4 до 8 людей. Она меняется, у нас бывает реорганизация, всякое такое, но в основном размер команды в пределах этого.  Теперь о том, как мы коммуницируем. Когда мы часто между собой разговариваем, мы можем проговаривать, например, такое: “Я делаю этот сервис уже 2 года, я бы хотел начать делать что-то другое. Вот у нас есть 3 сервиса, можно я перейду с этого на другой?” Но я не могу сказать: “Да, мне надоел этот сервис, давайте напишем какой-то пятый”. Бывают случаи, когда мы действительно приходим к пониманию, что нам нужен пятый сервис, который будет заниматься какими-то другими вещами, и тогда мы артикулируем, что именно он будет делать. Есть дизайн-сессии, на которых  кто-то презентует своё виденье, все остальные с ним соглашаются или не соглашаются, и мы приходим к какому-то общему решению, которое все поддерживают. 

![Meetup Photo 3]({{ "/img/meetup_netflix/photo_3.jpg" | absolute_url }}){: .in-post-img-center .in-post-img-border}

**ДМ**: Ок. Как вы снимаете аналитику после выполнения какой-нибудь стратегической задачи? Кто этим занимается? Вы сами или есть специальные роли, например, департамент аналитиков, который следит за этим? 

**АК**: Такого нет. Это задача всей иерархии менеджеров, и наша в том числе.

**ДМ**: То есть, вы самодостаточные и можете сами проверить что-либо?

**АК**: Нет, мы не можем проверить что-либо, но частично моя задача заключается в том, чтобы проверить мою идею с пользователями. Львиная доля идёт на то, чтобы ходить к другим командам и спрашивать их: “Ребята, что сделать, чтобы вам было лучше? А если я это сделаю? А если это?” Очень часто мне накидывают много разных требований, я прихожу к какому-то общему знаменателю, и говорю: “Ок, если я сделаю это, оно поможет многим людям одновременно”. 

**ДМ**: Хорошо. Есть ли у вас какая-то выделенная роль архитектора? Или application архитектора, или solution архитектора?

**АК**: Нет. У нас нет тайтлов. Каждый, кто работает на Netflix, это senior software engineer. При всём при этом, когда собирается комната, и там написано 6-8 людей, которые все занимаются архитектурой, ты в какой-то момент начинаешь понимать, кто из этих людей на много голов старше меня, потому что этот человек в голове держит то, как функционирует много разных систем, он может понять, как одна влияет на другую, и когда он ставит вопрос, этот вопрос такого уровня сложности, на который нелегко ответить. Ты это просто понимаешь.

**ДМ**: Человек так же пишет код, как и другие члены команды?

**АК**: Да. Так или иначе, но уровень seniority проявляется, просто он не формализованный, и он не даёт никому права, даже если ты молодой-зелёный, говорить: “Да ты молодой-зелёный и ничего ещё не шаришь”. На любой вопрос, который я задам, мне дадут ответ. Это может быть ответ типа “мы уже пару раз объясняли, даже написали док, вот тебе линк, иди посмотри”. А может быть “да, это сложный вопрос, мы над ним бьемся уже несколько месяцев, пока что подходы есть такие и такие, что ты предлагаешь?” 

**ДМ**: Ок. А есть ли жёсткая конкуренция между людьми внутри команды, если они все синьоры и имеют одинаковые тайтлы? Какая-то иерархия выстраивается или нет? И как менеджит это сама команда? Ведь это может как разрушить команду, так и дать какие-то бенефиты.    

**АК**: Может. Но опять-таки, мы приходим к определённой коммуникации. Если у вас есть доверие друг к другу, что вы не против друг друга, а заодно, что вы все строите общую систему и у вас общий успех, то намного проще в таком случае доверять суждениям людей, их вопросам, критике.

**ДМ**: Наверное, это до момента, пока не начнут запрашивать райзы или упираться в какие-то вещи, связанные с доходом? Как-то же нужно понять, кто в команде, например, заслуживает райза, а кто нет?

**АК**: Я не могу ответить на этот вопрос. Я этими вопросами не заморачиваюсь, потому что это не моя роль, понимаешь. И я не чувствую на себе того негативного эффекта, который ты описываешь. Я некомпетентен в этом вопросе. Возможно, в Netflix кто-то такое видел или чувствовал, но это не мой опыт. 

**ДМ**: А у тебя есть какой-то personal development plan?

**АК**: Да, но он мой личный, то есть, я его составил сам для себя. Не менеджер, а я сам, пришёл и сказал, что я хочу больше заниматься public speaking. Мне сказали: “Хорошо”. Я сказал, что поеду для этого в Украину. Мне сказали: “Хорошо”. (смех зала)

**ДМ**: А оценки 360?

**АК**: Да. Они есть. 360 происходит раз в год, довольно формализованная вещь. Ты можешь написать фидбек всем, запросить его у всех. Нет чёткой структуры, что писать, есть рекомендации. Например, что писать по паттерну start-stop-continue - это начать, это перестать, а это продолжить. Есть паттерн situation-behaviour impact - где ты описываешь ситуацию, каким было поведение, и какие это имело последствия. Но всё в целом сводится к тому, что ты пытаешься дать людям не оценку, а контекст. То есть, описать, что ты видишь, вместо того, чтобы говорить, хорошие они или плохие. 

**ДМ**: Это как-то решает проблемы в командах или нет? По моему мнению, если в команде есть синьоры, взрослые люди, они могут пообщаться и договориться, то есть, не должно быть “детского сада”. Например, какой-то условный Вася греет в офисной микроволновке рыбу, стоит запах на весь офис. Эти вещи можно решить, пообщавшись один-на-один. В некоторых украинских офисах это выносится на осуждение публики, проводят ван-ту-ван, приобщают HR-департамент, разных менеджеров, проводят поучительную работу, мол не делай так, а потом ещё и ретроспективу - делал или нет. (смех зала)

**АК**: Ну, если столько ресурсов тратится на то, чтобы решить вопрос с микроволновкой, я надеюсь, что у вас остаётся время на то, чтобы быть эффективными и зарабатывать деньги. 

**ДМ**: Это просто как пример. Какого уровня проблемы могут всплывать у вас на 360? 

**АК**: Могут быть вопросы коммуникации между командами. Если мы две команды, которые взаимодействуют, но мы в разных организациях, у нас разная внутренняя структура, иногда необходимо как-то эти вопросы проговаривать, и, если эти вопросы недостаточно артикулированы, ты можешь вспомнить о них в 360. Понимаешь, 360 это такая ретроспектива того, что произошло за год, возможность вспомнить всё хорошее и плохое, как перед Новым годом.

**ДМ**: Можешь привести примеры вопросов о коммуникации с другими командами, которые вы выносили в 360? Это проблемы с персоналом, кто-то не friendly, не помог в чём-то или что-то другое?

**АК**:  Погоди, вопрос того, что кто-то не friendly это вопрос номер 0. Если человек не может быть вежливым и приятным в общении, ему очень сложно построить любые связи. На идею brilliant jerks Netflix смотрит с непониманием. Brilliant jerks это когда человек гениальный технический специалист, но при этом у него нет эмпатии и он не проявляет уважения к другим. И это очень часто мешает коммуникации между разными людьми. Вообще в этом плане я очень рекомендую почитать “Project Galileo”, сделанный Google, о том, что мы знаем, что делает эффективным индивида, но не знаем, что делает эффективной команду. У них есть две статьи о том, как строить эффективную команду, лично мне они очень нравятся.

**ДМ**: Есть ли вопросы у зала? 

**Зал**: Как происходит пересмотр зарплаты?

**АК**: Раз в год.

**Зал**: По каким принципам? Если нет чётких KPI, если ты сам делаешь себе метрики в Jira, как происходит процесс пересмотра зарплаты?

**АК**: Очень просто. Сколько у тебя пользователей? Сколько команд зависит от тебя? Скольким людям ты помогаешь? Сколько команд пользуются услугами твоего сервиса? 

**Зал**: 5

**АК**: Ок, а до этого сколько было?

**Зал**: 4

**АК**: Вот, значит, стало на 1 больше.

**Зал**: Как это повлияет на зарплату?

**АК**: Ну, опять-таки, как это конкретно повлияет на зарплату в конкретной компании, зависит от вашего разговора и вашей компании. Если вдруг до этого пользовались 3 маленькие команды, а теперь это одна фундаментальная команда, и от твоего сервиса зависит работа всего бизнеса, это важный сервис, ты делаешь что-то, что очень влияет на бизнес...такой человек, наверное, очень ценен для компании, и мне это кажется логичным.

**ДМ**: Кстати, а за факапы как-то наказывают?

**АК**: Опиши понятие “факап”.

**ДМ**: Зарелизили какой-то сервис, а он нагнулся от нагрузки.

**АК**: А если потом поднялся?

**ДМ**: Ну, баги, нестабильно работает.

**АК**: Я не менеджер, поэтому я не знаю, как это оценить. Но из того, что ты описываешь, я думаю, что тут банальный вопрос технической компетенции. 

**Зал**: Мы пытаемся просто понять процессы пересмотра зарплаты. 

**АК**: Человека увольняют в таком случае (смех зала). Я чего-то не понимаю?

**Зал**: Это утрированный пример. Мы пытаемся понять процесс пересмотра. Чем руководствуются? Вот ты приходишь и говоришь: “Я хочу +50”, или как?

**АК**: Ну смотри, в Netflix есть такой момент. Если человек говорит: “Я хочу +50”, то они пытаются дать ответ на 3 вопроса: 1) что компания потеряет, если этот человек уйдёт? 2) сколько в среднем люди такого уровня получают? 3) сколько будет стоить компании нанять человека с похожим скилл-сетом на эту позицию? И часто мы говорим только про вопрос номер 1 - сколько я буду получать в компании или вообще в индустрии? Если кто-то получает в индустрии +200%, но для того, чтобы нанять человека на это место выполнять эту работу, ты можешь заплатить те же деньги и быстро нанять другого специалиста, сложно держать такого человека. Ты можешь зарабатывать больше. Возможно, у тебя есть скиллы, которые ещё не используются. 

**ДМ**: Единый рынок прайсов. 

**АК**: Типа того. 

![Meetup Photo 4]({{ "/img/meetup_netflix/photo_4.jpg" | absolute_url }}){: .in-post-img-center .in-post-img-border}

**Зал**: Работая в Нетфликсе, были ли какие-то инсайты, после которых появляется inspiration на на создание своего стартапа и была ли вообще мысль про создание своего стартапа и своего бизнеса?

**АК**: Когда ты в Силиконовой Долине, ты подхватываешь это как грипп (смех зала), и носишься постоянно с этими продуктами и идеями. Да, эти идеи в воздухе, и постоянно есть желание что-то своё интересное сделать, но ты реально понимаешь, что это масса пота, крови и недоспанных ночей, а не просто смузи и Старбакс. Это реально сложно, и , опять же, когда ты в Долине, ты видишь не только людей, которые взлетели, но и это кладбище людей, которые вложили львиную долю своей жизни и здоровья для того, чтобы что-то получилось. Когда ты это видишь, ты не уверен, что реально готов. Но идеи есть.

**Зал**: За два года в Netflix вы ходили на какие-то собеседования?

**АК**: Netflix относится к этому довольно просто - если ты хочешь пойти проверить, сколько тебе могут предложить, ты можешь. Но идея такая, что компания не будет тебя сдерживать от того, чтобы поменять работу. Но, если единственным основанием для смены работы является зарплата, то они хотят, чтобы это было где-то внизу списка твоих приоритетов.То есть, если ты хочешь быть VP of engineering, а тут ты обычный программист, может есть смысл идти в стартап. Но идея такая, что нужно знать, что ты сам хочешь 

**Зал**: И Вы ходили за два года на собеседования?

**АК**: Почему это важно? Что это даст?

**Зал**: Интересно, присматриваетесь ли вы к другим компаниям?    

**АК**: Ну, смотрите, количество рекрутёров, которые стучат мне в инбокс - почти константа. И это никак не повлияет на мою работу на компанию. 

**Зал**: Мне это даст понимание, как Вы смотрите на индустрию, и ходите ли вы смотреть, что другие компании делают, что предлагают, куда это движется.   

**АК**: Что делают другие компании, я узнаю путём участия в форумах, митапах. Меня интересует video dev. Я хожу на пару митапов в Сан-Франциско, пару митапов местных, и общаюсь с людьми, которых я знаю по индустрии.  

**Зал**: Какие скилы Вы можете подкачать?

**АК**: Это происходит путём общения с инженерами. Интервьюирование, на самом деле, это довольно затратный процесс, который предполагает мою подготовку в том числе. Мне нужно найти на это время, сконцентрироваться. Поэтому я не уверен, что это самый эффективный способ получить ответ на Ваш вопрос.

**Зал**: Как вы шарите знания в компании между командами? Например, есть компания, которая что-то делает, и другой это могло бы быть полезно, но они об этом просто не знают. Как донести знание, что вы сделали что-то крутое, до всех команд, до другой компании? Как происходит процесс передачи знаний?

**АК**: Это не формализованный процесс, очень помогают странички вики, гуглдоки. К сожалению, ты не можешь построить унификованный способ шеринга информации, и при этом ожидать, что все команды будут иметь freedom & responsibility, что у каждой команды будет своя свобода. Поэтому это очень часто командное усилие - я общаюсь с другими людьми, узнаю, что они делают, и потом делюсь этой информацией внутри команды. И так делает почти каждый участник, и так делает наш менеджер, менеджер менеджера, и так далее. Они знают, что мы делаем, когда они видят что-то похожее, они приходят и говорят: “Слушайте, мы услышали, что там делают какую-то интересную штуку, давайте об этом поговорим”. И часто бывает, что развиваются два похожих продукта, но выигрывает лучший. и один переезжает на другой, или наоборот.

**Зал**: Бывают конференции или презентации каких-то ключевых решений внутри компании?

**АК**: Да. Такие митапы называются brown bag. Они проводятся с определённой каденцией. Объявляется наперёд, о чём будут говорить, а также есть рассылка на всех, кто подписан на эти штуки. Они бывают на разных уровнях - меньшей организации, большей организации, в зависимости от того, кто в чём заинтересован.

**Зал**: У меня есть вопрос по взаимодействию. В интернете читал про Netflix - есть программисты, DevOps, QA, весь набор, и, когда формируется команда, иерархия или knowledge sharing идёт отдельно - отдельно программисты, у них есть главный менеджер, отдельно девопс, и они культуру распределяют среди девопсов, и каждый девопс в команде рассказывает, как мы реализуем что-то. Есть ли это?

**АК**: То есть, есть ли распределение между программистом и девопсом? Я такого не видел. Я мейнтейнлю все свои деплойменты, я ответственный за все сервисы, которые я сделал, и также ответственный за другие сервисы, которые со мной взаимодействуют, за другие вещи, которыми занимается моя команда.

**Зал**: А инфраструктурой тогда кто занимается? Просто отдельная команда, которая предоставляет тебе сервис?

**АК**: Типа того.

**Зал**: То есть, есть одна группа людей, например, девопс, и он в роли девопса, и эта роль по-разному распределяется. Такого нет?

**АК**: Нет. То есть, идея такая, что каждая команда пытается свою работу подать как платформу. Если я делаю базу данных, я пытаюсь делать так, чтобы этой базой данных могло наибольшее количество людей воспользоваться. Если команда занимается AWS, они делают так, чтобы они могли большему количеству команд, которые выиграют от того, что будут пользоваться AWS. Если команда занимается security, то их роль заключается в том, чтобы security можно было использовать как можно проще.

**Зал**: Грубо говоря, если мы берём engineering иерархию, то везде это универсальный боец?

**АК**: Наоборот.

**Зал**: Специализированный?

**АК**: Мы наверное заходим в разговор “fullstack vs specified”?

**Зал**: Нет, если мы говорим о fullstack, то это как раз от архитектора до девопса, до деливери. То есть, от идеи до имплементации - вот это fullstack. А не фронт и бэк. То есть, это универсальный программист, который умеет спроектировать и умеет задеплоить.

**АК**: Такого нет.

**Зал**: То есть, есть отдельно тот, кто проектирует, отдельно тот, кто разрабатывает и отдельно тот, кто деплоит?

**АК**: Такого тоже нет. (смех зала) Я проектирую как команда. Я девелоплю как индивид. Я деплою как индивид в координации с командой с другими сервисами. Я деплою при помощи тулзов, которые предоставлены мне другими командами. Есть команда, которая отвечает за нашу build систему, есть команда, которая отвечает за management & EC2 instances, есть команда, которая отвечает за предоставление нам баз данных, есть команда, которая отвечает за другие аспекты. При этом всём я пользуюсь всей их помощью, поддержкой, но, если у меня возникают какие-то проблемы, я в первую очередь сам решаю, что это проблема, и когда я натыкаюсь на полное непонимание, стену, и понимаю, что я потрачу на это 2 дня, то лучше пойти в отдельный slack канал и попросить помощи.

**Зал**: Из этих команд строится одна универсальная, которая работает над сервисом, или нет?

**АК**: Одна универсальная...Все эти команды предоставляют свои определённые сервисы, и я пользуюсь их сервисами для того, чтобы сделать свой.

**Зал**: То есть, команды чётко сформулированные и они предоставляют сервисы?

**АК**: Да.

> На этом вторую часть закончим. Последняя часть будет скоро!

А чтобы не пропустить следующие части, подписывайся на мой авторский telegram канал
[«Об IT без галстуков»](https://goo.gl/E3AFL1). На канале, с позиции CTO продуктовой компании, я делюсь своим
видением технологий, пишу техничку, про менеджмент персонала, планы личностного развития и психологию построения
команд.

Если Вы готовы к серьезных челенджам и разделяете подход freedom and responsibility, готовы в релокейту в USA, то
референс в netflix запросить можно у Арсена по адресу `arsen.kostenko at gmail`. Я тоже разделяю подобные подходы
и внедряю их в Авроре, еще и сфера деятельности совпадают. Экспертов желающих поработать со мной в Киеве рад видеть и
у себя. Пишите на `d.menshikov at gmail`.

> Кстати, про свои и командные фейлы с винами я рассказывал в
[публикации про предновогодний релиз]({{ site.baseurl }}{% link _posts/2019-10-05-story-of-one-ny-release.md %})!
Это пример того что мы делаем и с чем сталкиваемся, а не эти всякие смузипроводы, формошлепсткие галеры и прожигание
жизни без цели. Мы же решаем проблемы, имеем стратегию и знаем куда идем.