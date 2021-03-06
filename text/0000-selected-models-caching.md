- Дата создания: 2018-02-12
- RFC PR:
- RFC Issue:
- Flexberry Issue:

# Кэширование выбранных моделей на клиенте

## Краткое описание

В `ember-flexberry-data` уже есть механизм офлайн-моделей, данные которых 
хранятся на клиенте. Предлагается добавить сервисы по синхронизации локальных
данных с данными на сервере.

## Обоснование

В ряде прикладных проектов есть необходимость кэшировать редко изменяемые модели, например, 
справочники. Причина - небыстрые каналы связи или пожелания заказчика. Имея данные 
локально мы добьёмся предположительно более отзывчивой работы приложения и снижения количества
запросов на сервер бэкенда.

## Детальное проектирование

В аддоне `ember-flexberry-data` есть поддержка т.н. "офлайновых" моделей, т.е. 
тех, данные которых хранятся в локальном хранилище браузера (IndexedDB), и работа с которыми 
идет через `local-store`. Признак того, что модель является офлайновой может меняться 
динамически. Так же есть сервис `syncer`, который позволяет загружать данные модели в локальное
хранилище и отправлять изменения в бэкенд.

Предлагается реализовать сервис-загрузчик, который бы для указанного набора моделей 
выполнял загрузку данных модели из бэкенда в локальное хранилище. А также сервис-планировщик, запускающий первый сервис по расписанию или по сигналу бэкенда.

Для описания метаданных кэшируемого объекта добавим на клиенте модель
`offline-loading-object` со следующими полями: 
```js
// имя модели
modelName: DS.attr('string'),
// человекочитаемое название
caption: DS.attr('string'),
// размер порции загружаемых данных
loadCount: DS.attr('number'),
// дата, время последней загрузки
lastSyncDate: DS.attr('date'),
// дата, время последнего изменения
lastChangeDate: DS.attr('date'),
// дата, время следующей проверки необходимости загрузки
// например, вычисляется из lastSyncDate и checkPeriod
nextSyncDate: DS.attr('date'),
// как часто нужно проверять изменение данных, строковое значение: 1 hour, 1 day, 2 week, 1 year
checkPeriod: DS.attr('string'),
```
Она будет храниться в локальном хранилище и работать с ней будем в офлайн-режиме.

Далее в сервисе-загрузчике описываем массив загружаемых моделей: 
```js
...
loadingObjects: [{
    modelName: 'iis-eais-catalogs-vid-udost-dok',
    caption: 'Вид удостоверяющего документа',
    loadCount: 50,
    checkPeriod: '1 day'
}, {
    modelName: 'iis-eais-catalogs-tip-udost-dok',
    caption: 'Тип удостоверяющего документа',
    loadCount: 50,
    checkPeriod: '1 day'
}, {
...
```
> Для работы в офлайн-режиме указанные модели должны быть помечены как офлайновые в сервисе `store`,
и для них должна быть прописана схема для таблиц `dexiedb`.

Начиная работу сервис-загрузчик приводит в соответствие данные `offline-loading-object`, хранящиеся
в локальном хранилище, с тем, что описано в массиве `loadingObjects`. Затем для указанных моделей 
выполняется запрос даты их актуальности. (Через вызов метода webapi или odata-функции). В качестве
такой даты можно принять поле аудита модели - время изменения - брать самое последнее. Если данные на 
сервере новее и текущее время больше `nextSyncDate`, то сервис переключает модель для работы в 
онлайне, удаляет данные модели из локального хранилища, и скачивает новые из бэкенда порциями и 
загружает в локальное хранилище. (Можно использовать все поля модели, либо указывать необходимые 
через представление). Если операция завершилась успешно, модель помечается как офлайновая; 
если произошла ошибка, модель остается онлайновой (при следующем запуске сервис повторит попытку 
загрузки).

Запуск метода сервиса-загрузчика можно делать в обработчике некоторого повторяющегося действия, 
например, логина пользователя. Либо из сервиса-планировщика, который запускается с минимальным интервалом, например, 5 минут, и запускает сервис-загрузчик. А тот уже сам обрабатывает необходимость 
загрузки данных. 

Либо изменения можно получать по сигналу от бэкенда: реализуем сервис, который через SignalR или механизм long-polling получает пакет с изменениями от бэкенда и загружает их в локальное хранилище.

Изменения таких кэшированных моделей предлагается дублировать в бэкенд. Чтобы иметь там полностью 
актуальную версию данных.

## Документирование и обучение

1. Стандартная документация добавляемых сервисов.
2. Необходимо будет создать статью по предоставляемым возможностям и использованию 
этого решения в прикладных проектах.

## Недостатки

На клиенте будет возникать ситуация неактуальности кэшированных данных. Но возникновение такой
ошибки будет сигналом к обновлению кэша.

Функционал предлагаемого решения не всегда востребован на прикладных проектах.

## Альтернативы

Можно рассмотреть механизм кеширования браузера с помощью HTTP заголовков Cache-Control и т.д.
[Подробнее тут](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=en#cache-control).
Он требует добавления соответствующего обработчика на стороне бэкенда, который будет добавлять 
заголовок кеширования, валидировать кэш для выбранных запросов.  

Плюс: На клиенте используется стандартный механизм и требуется доработка только бэкенда.   
Минус: Избыточное кэширование запросов с наложенными ограничениями.

Также настройки кэширования в браузере может изменять пользователь, что нужно учитывать.

## Нерешенные вопросы

Уточнить:
* Как делать частичную синхронизацию данных модели.
* Возможность синхронизации модели по сигналу с сервера.
