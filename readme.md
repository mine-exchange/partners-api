# API

## Типичные сценарии работы 🔀

### Работа с заказами:
- [Создать заказ](#создание-заказов);
- Если выплата производится с MineCoin и не была подтверждена при создании заказа — [Подтвердить заказ с MineCoin](#подтверждение-заказов-с-minecoin). Однако, для ускорения работы, мы рекомендуем использовать параметр "[auto_confirm](#create_order_auto_confirm)" при создании заказа. Это исключает необходимость отдельного подтверждения заказов;
- Если необходимо [подтвердить оплату](#orderpayment_variants--возможные-способы-оплаты-заказа-клиентом-используется-в-ответах) - [Подтвердить оплату заказа](#подтверждение-оплаты-по-заказам);
- Проверка статуса, обычно [по webhook уведомлению](#webhook-уведомления-о-статусах-заказов-), однако в некоторых случаях можно сделать явный запрос для [получения состояния группы заказов](#получение-заказов);
- Иногда требуется [удалить заказ](#отмена-заказов), пока он не пошёл в работу.

### Иногда требуется уточнить:
- [Курсы, ограничения, комиссии](#получение-текущих-курсов-ограничений-минимальноймаксимальной-суммы-и-резервов)
- [Балансы MineCoin счетов](#получение-балансов-minecoin)
---

## ⚡️ Быстрый старт (коллекция методов с автоматизацией подписи в Postman) 

Для вашего удобства мы создали коллекцию в [postman](https://www.postman.com/), которая повторяет все методы API. Вы можете потестировать на ней функциональность и поиграть с запросами.
1. В postman выбрать пункт меню "Import file", например в вкладке "Home"
2. Перейти в вкладку "Link" и импортировать следующую ссылку: ```https://api.postman.com/collections/21262179-a7496b21-df82-43f6-aae8-74b48e3b49e6?access_key=PMAT-01H83YE21HHE2RAA4APRH61SJZ```
3. Перейти курсором на коллекцию "MINE Partners API", выбрать вкладку "Pre-request script" и вписать ваши ключи в блоке "Configuration"

---

## ⚠️ Финансовый контроль
Один из необычных аспектов, на которые следует обратить внимание это когда при создании заказов указывается не опалчиваемая сумма а сумма, которая будет выплачена. Дело в том, что процессор нашей системы рассчитывает сумму к выплате на основе указанной суммы к приёму. Результат может попасть на дробный шаг, который округлится на копейку. В результате может быть расхождение от запрошенной вами суммы к выплате на какие-то мелкие доли коппек. Поэтому для строгого учёта следует после создания заявки актуализировать суммы в своей БД во избежание нанорасхождений.

Также на некоторых направлениях действует пересчёт заказов. Это значит что в случае поступления неверной суммы от клиента, заказ пересчитывается и выполняется исходя из суммы поступления. В момент получения финального статуса заказа (предпочтительно по webhook), необходимо учитывать суммы в ответе API и учитывать именно их в вашей системе. Недопустимо доверять зачислению без сверки сумм. 

Также, рекомендуем регулярно сверять балансы от нашего API и вашей системы.


## ⚠️ Контроль сетевых потерь
Связь между серверами хоть и редко, но может обрываться, поэтому необходимо валидировать, являются ли полученные в ответ данные ожидаемыми вами:
1. Если запрос прошёл корректно, клиент получает 2xx http код. В иных случаях произошла ошибка запроса, а [в теле ответа содержатся её подробности](#стандартный-ответ-об-ошибке-для-всех-запросов). Альтернативный вариант проверки прохождения запроса — проверка параметра success в json объекте тела ответа (true если запрос прошёл, иначе false).
2. API всегда отвечает JSON ответом. Если вы не получили корректный JSON — вы не можете точно знать, прошёл ли ваш запрос или был потерян по пути. В случае критичных запросов (например, выплата с автоподтверждением), следует перепроверять такие запросы любым доступным вам способом и не считать запрос выполненным или не выполненным, пока не получите точного подтверждения произошедшего. 
    - В некоторых случаях вы можете отправить уточняющий API запрос для проверки состояния. Например, если вы делали запрос на выплату и не знаете, прошёл ли он, вы можете [проверить существование и состояние заказа](#получение-заказов).
    - В случае невозможности автоматизированной проверки, следует обратиться в техподдержку. Наша поддержка действительно оперативна, поэтому при возникновении такой исключительной ситуации, вы получите ответ максимально быстро 💪

## Безопасность 🛡

Для обеспечения безопасности используются 2 ключа: ключ авторизации и ключ подписи.
Оба ключа генерируются пользователем в личном кабинете в вкладке "API настройки".

### API авторизация
Для авторизации в API протоколе используются ключ авторизации, который необходимо передать как Header:
```js
Authorization: "Bearer {{AuthKey}}"
```

### Подпись API запросов
В целях безопасности в запросах к API требуется подпись, которая хеширует тело запроса с помощью ключа подписи.
Сгенерированную подпись необходимо передать как Header: 
```js
X-Signature: "{{Signature}}"
```

Сама подпись генерируется по алгоритму: 
```php
hmac(sha256(SignatureKey, Request.Body))
```
(Где SignatureKey — ключ подписи, а Request.Body - тело конкретного отправляемого запроса).

Если тело запроса отсутствует (GET запросы), подпись делается от пустой строки.

📢 Аналогичный [подход к подписыванию запросов](#проверка-подлинности-webhook-уведомлений) используется и при отправке webhook уведомлений с нашей стороны. Разница лишь в том, что в случае API запроса от вас к нам, вы генерируете подпись со своей стороны, а мы её проверяем. При отправке же webhook уведомлений от нас к вам, подпись генерируется нами, а проверяете её вы.

⚡️ Для быстрого тестирования, рекомендуем воспользоваться [нашей коллекцией в Postman](#%EF%B8%8F-быстрый-старт-коллекция-методов-с-автоматизацией-подписи-в-postman)


### Разрешенные IP-адреса
Для большей безопасности, доступ к методам API разрешён только с определённых IP-адресов (назначаются в личном кабинете в вкладке "API настройки"). 

⚠️ Вы можете указать символ "**\***" для обеспечения свободного доступа со всех IP адресов, но мы **настоятельно не рекомендуем это делать на production**.

---
## Заголовки для всех API запросов
```json
{
    "Accept": "application/json",
    "Content-Type": "application/json",
    "Authorization": "Bearer {{auth_key}}",
    "X-Signature": "{{signature}}"
}
```

## Стандартный ответ об ошибке для всех запросов:
```json
{
    "success": false,
    "code": "4xx-5xx",
    "errors": {
        "message": "{{error_message}}",
        "details": {}
    }
}
```

## Схема объекта "Order" (используется во множестве API запросов и ответов)
```js
{
    "order_id": "{{ID заказа в системе партнера}}",
    "core_system_order_id": "{{ID в ядре (в нашей системе); Используется в API ответах}}",
    "inbound": {
        "currency": "{{Валюта отправки}}",
        "account": "{{Аккаунт отправителя}}",
        "amount": "{{Сумма отправки}}",
        "amount_initial": "{{Изначальная сумма отправки; Используется в API ответах}}",
        "txid": "{{ID входящей транзакции; Используется в API ответах}}"    
    },
    "outbound": {
        "currency": "{{Валюта выплаты}}",
        "account": "{{Аккаунт получателя}}",
        "amount": "{{Сумма выплаты}}",
        "amount_initial": "{{Изначальная сумма выплаты; Используется в API ответах}}",
        "txid": "{{ID исходящей транзакции; Используется в API ответах}}"
    },
    "comment": "{{Комментарий к заказу (необязателен)}}",
    "status": "{{Статус операции: new|processing|incoming_unconfirmed|incoming_confirmed|moderation|error|canceled|success; Используется в API ответах}}",
    "created_at": "{{Время создания в формате ISO 8601; Используется в API ответах}}",
    "updated_at": "{{Время изменения в формате ISO 8601; Используется в API ответах}}"
}
```

### Что такое amount_initial?

При [инициации заказа](#создание-заказов) amount_initial не передаётся. Указывать требуется outbound.amount или inbound.amount, который сразу зафиксируется в amount_initial

Причины почему amount и amount_initial могут отличаться:
- Был создан заказ CARDUAH -> BTC, на 0.1 BTC, в результате рассчёта получилось, что нужно оплатить 100к грн, но было оплачено только 50к, движок пересчитал заявку и вылатил 0.05 BTC
- Иногда могут поплыть копейки из за подкапотных пересчётов.

### Возможные значения параметра status

| **status** | **Описание** |
|--|--|
| **new** | Новый заказ, помещён в очередь, не обработан. |
| **processing** | Заказ помещён в обработку, ожидает действий от пользователя (оплаты или подтверждения с MineCoin). |
| **incoming_unconfirmed** | В некоторых случаях после оплаты клиентом система видит транзакцию в сети, однако средства ещё не поступили и попытка может провалиться. В этом случае ставится этот статус, который подразумевает ожидание входящего платежа. Также данный статус устанавливается после успешного [подтверждения оплаты](#подтверждение-оплаты-по-заказам). |
| **incoming_confirmed** | Платёж от клиента подтверждён. |
| **moderation** | Заказ проходит модерацию менеджера сервиса. Редкий статус. |
| ❓ **error** | Ошибка в заказе, статус не финальный, свяжитесь с техподдержкой для уточнения ситуации. |
| 🏁 **canceled** | Заказ отменён (пользователем или системой). В направлениях c MineCoin данный статус означает, что заказ получил финальный статус, деньги по заказу не отправлены и возвращены на счет MineCoin. Возможные причины: ошибка в реквизитах получателя, ограничения банка эмитента, иные причины. После получения этого статуса можно повторять отправку. |
| 🏁 **success** | Выплата средств произведена, заказ выплачен. |


🏁 — Финальные статусы. 

❓ **error** — Произошла неизвестная исключительная ситуация, требуется связаться с поддержкой для уточнения деталей. Повторную отправку делать нельзя.

---


### Необязательные, но возможные параметры, дополняющие информацию о заказах:

#### Order.extra — Дополнительные данные, которые партнёр мог передать при создании заказа (используется в Запросах и Ответах)
```js
{
    "extra": { 
        "client": {
            "id": "{{ID пользователя в вашей системе}}",
            "email": "{{Email адрес пользователя вашей системы}}",
            "first_name": "{{Имя пользователя вашей системы}}",
            "last_name": "{{Фамилия пользователя вашей системы}}",
            "middle_name": "{{Отчество пользователя вашей системы}}",
            "ip": "{{IP-адрес пользователя вашей системы, который совершает действие}}",
            "user_agent": "{{Useragent (браузер) пользователя. Примечание для php разработчика: содержится в $_SERVER['HTTP_USER_AGENT']}}"
        }
    }
}
```

#### Order.payment_variants — Возможные способы оплаты заказа клиентом (используется в Ответах)
```js
{
    "payment_variants": {
        "invoice": {
            "url": "{{https://***}}",
            "error": null|"{{Описание ошибки}}"
        },
        "direct": {
            "url": "{{https://***}}",
            "error": null|"{{Описание ошибки}}"
        },
        "manual": {
            "number": "{{***}}",
            "error": null|"{{Описание ошибки}}",
            "memo": null|"{{memo}}",
            "is_confirmation_needed": true|false
        }
    }
}
```   

**Реквизиты payment_variants в зависимости от приёмной платёжной системы**

⚠️ Данное описание реквизитов актуально на момент написания и служит исключительно для ориентира интегратора. Набор параметров может измениться.

| **Платёжная система** | **Объект в payment_variants** | **Описание** |
|:--|:--|:--|
| **BITCOIN** (BTC)                         | manual  | (number) Номер btc кошелька для оплаты |
| **Остальные мерчанты**                    | direct  | (url) Ссылка на платёжную форму мерчанта |    

**Каждый элемент объекта "payment_variants" содержит следующие параметры:**
- error (Обязательный параметр. По умолчанию null, нет ошибки. В противном случае текстовое описание ошибки, из за которой невозможно использование данного метода оплаты).
- Один из следующих реквизитов оплаты, обязательный параметр:
    - url: адрес платёжной системы, куда необходимо переадресовать клиента. 
    - number: реквизиты, по которым клиенту необходимо вручную перевести оплату.    
- Дополнительные параметры для ручной(manual) оплаты:
    - memo: дополнительный идентификатор транзакции, требуется для некоторых криптовалют.
    - is_confirmation_needed: флаг необходимости подтверждения оплаты. При значении `true` - необходимо [подтвердить оплату](#подтверждение-оплаты-по-заказам).
- note (Текстовое примечание для упрощения интеграции программистам и упрощения процесса интеграции. Необязательный параметр, иногда присутствующий).





---
## Методы API 🔢

### Получение заказов

⚠️ Мы не рекомендуем регулярно пользоваться данным методом во избежание задержек. Для уведомлений о изменениях состояния заказов реализован инструмент "[Webhook уведомления о заказах](#webhook-уведомления-о-статусах-заказов-)"

**[GET] /{API_PREFIX}/orders?ids={{id1}},{{id2}},...&page={{pages_count}}&per_page={{items_count}}**
- ids: перечисленные через запятую номера заказов партнёра. Допустимо указать один конкретный заказ;
- page: необязательный параметр, страница списка, которую требуется получить; 
- per_page: количество элементов на 1 страницу;

#### Успешный ответ (200)
```js
{
    "success": true,
    "code": 200,
    "data": {
        "total": {{Общее количество элементов в выборке}},
        "per_page": {{Количество элементов на страницe}},
        "page": {{Текущая страница}},
        "orders": [
            "{{ID заказа в системе партнера}}": {{Order}},
            ...
        ]
    }
}
```
ℹ️ Смотрите также: [Описание cхемы объекта "Order"](#схема-объекта-order-используется-во-множестве-api-запросов-и-ответов).

⚠️ В случае, если поиск статусов заказов не найдёт ни одного заказа по вашим параметрам запроса, в параметре ответа data.orders вы получите пустой объект. Поэтому если вы пользуетесь данным методом для переопроса состояния заказов (что делать крайне нежелательно но относимся с пониманием к вариантам различных бизнес-задач), вам следует проверять наличие ваших заказов в ключах элементов объекта data.orders. 


---
### Создание заказов

Данный запрос создаёт заказы, но не реализует непосредственно оплату (кроме явного [автоподтверждения при создании заказа с MineCoin](#create_order_auto_confirm)). После создания заказа, для его выполнения необходимо [совершить оплату по реквизитам](#orderpayment_variants--возможные-способы-оплаты-заказа-клиентом-используется-в-ответах), выданным в ответе (или [подтвердить намерение в случае выплаты с MineCoin](#подтверждение-заказов-с-minecoin))

Для некоторых платежных систем, при оплате по реквизитам, требуется подтверждение оплаты. Если в реквизитах payment_variants флаг [is_confirmation_needed](#orderpayment_variants--возможные-способы-оплаты-заказа-клиентом-используется-в-ответах) установлен в true - после произведения оплаты её необходимо [подтвердить](#подтверждение-оплаты-по-заказам).

**[POST] /{API_PREFIX}/orders**
```js
{
    "orders": [
        {{Order}},
        ...
    ]
}
```
ℹ️ Смотрите также: [Описание cхемы объекта "Order"](#схема-объекта-order-используется-во-множестве-api-запросов-и-ответов).

**Обратите внимание: **
1. Невозможно создание заказа с одинаковым ID от одного партнёра.
2. При создании заказов поле inbound.account в некоторых случаях не является обязательным, например в случае выплаты с MineCoin.
3. Допустимо указание только одной из сумм — суммы к получению или суммы к выплате (inbound.amount или outbound.amount). Вторая сумма рассчитывается автоматически исходя из курсов и комиссий.
4. Вы можете указать поле "comment" для вашего удобства, однако получатель его не увидит. Данное поле будет отображаться в информации о заказе по API и в личном кабинете.

💡 Если у вас особый кейс, в котором не обязательно, чтобы ID заказа соответствовал ID в вашей системе, API может сгенерировать {{order_id}} заказа случайным образом. Для этого укажите его следующим образом:
```js
{
    "order_id": "auto-uid",
}
```

<span id="create_order_auto_confirm">💡</span> Если вы делаете заказ на выплату с MineCoin и требуется моментальная постановка заказа в работу, вы можете воспользоваться следующим параметром:
```js
{
     "auto_confirm": true
}
```
⚠️ Будьте осторожны, поскольку при этом произойдёт моментальное списание средств с вашего баланса и быстрая выплата средств на реквизиты получателя (теоретически, в случае ошибки вы можете успеть [сделать отмену](#отмена-заказов), но гарантии этого невозможны).


#### Успешный ответ (200)
```js
{
    "success": true,
    "code": 200,
    "data": {
        "created": {
            "{{ID заказа партнера}}": {{Order}},
            ...
        },
        "invalid": {
            "{{ID заказа партнера}}": "{{Сообщение об ошибке}}",
            ...
        }
    }
}
```


---
### Подтверждение заказов с MineCoin

ℹ️ Допустимо подтверждение заказов только в статусе "processing". 

**[PUT] /{API_PREFIX}/orders**

```js
{
    "ids": [
        "{{ID заказа партнера}}",
        ...
    ]
}
```

#### Успешный ответ (200)
```js
{
    "success": true,
    "code": 200,
    "data": {
        "confirmed": {
            "{{ID заказа партнера}}": {{Order}}
            ...
        },
        "invalid": {
            "{{ID заказа партнера}}": "{{Текстовое представление причины, по которой не удалось выполнить подтверждение}}",
            ...
        }
    }
}
```
ℹ️ Смотрите также: [Описание cхемы объекта "Order"](#схема-объекта-order-используется-во-множестве-api-запросов-и-ответов).

---
### Подтверждение оплаты заказов

ℹ️ Допустимо подтверждение заказов только в статусе "processing".

**[PUT] /{API_PREFIX}/orders/confirm-by-customer**

```js
{
    "ids": [
        "{{ID заказа партнера}}",
        ...
    ]
}
```

#### Успешный ответ (200)
```js
{
    "success": true,
    "code": 200,
    "data": {
        "confirmed": {
            "{{ID заказа партнера}}": {{Order}}
            ...
        },
        "invalid": {
            "{{ID заказа партнера}}": "{{Текстовое представление причины, по которой не удалось выполнить подтверждение}}",
            ...
        }
    }
}
```
ℹ️ Смотрите также: [Описание cхемы объекта "Order"](#схема-объекта-order-используется-во-множестве-api-запросов-и-ответов).

---
### Отмена заказов
В некоторых критичных случаях партнёрам бывает необходимо отменить созданные заказы, пока не начался процесс выплаты.

ℹ️ Допустимо подтверждение заказов только  в статусе "new" и "processing". Если начался процесс обработки оплаты, отмена этим методом API невозможна. В этом случае рекомендуем немедленно обратиться к нашим операторам.

**[DELETE] /{API_PREFIX}/orders**
```js
{
    "ids": [
        "{{ID заказа партнера}}",
        ...
    ]
}
```

#### Успешный ответ (200)
```js
{
    "success": true,
    "code": 200,
    "data": {
        "сanceled": {
            "{{ID заказа партнера}}": {{Order}},        
            ...
        },
        "invalid": {
            "{{ID заказа партнера}}": "{{Сообщение об ошибке}}",
            ...
        }
    }
}
```
ℹ️ Смотрите также: [Описание cхемы объекта "Order"](#схема-объекта-order-используется-во-множестве-api-запросов-и-ответов).


---
### Получение балансов MineCoin
Балансы внутреннего счёта партнёра в нашей системе (каждый счёт привязан к определённой валюте).

**[GET] /{API_PREFIX}/balances**

#### Успешный ответ (200)
```js
{
    "success": true,
    "code": 200,
    "data": {
        "BTC": {{Количество средств на балансе}},        
        "UAH": {{Количество средств на балансе}},
        "USD": {{Количество средств на балансе}}
    }
}
```


---
### Получение текущих курсов, ограничений минимальной/максимальной суммы и резервов

**[GET] /{API_PREFIX}/rates?directions={{CurrencyA}}:{{CurrencyB}},{{CurrencyX}}:{{CurrencyY}}**

#### Успешный ответ (200)
```js
{
    "success": true,
    "code": 200,
    "data": {
        "rates": {
            "{{CurrencyA}}:{{CurrencyB}}": {
                "inbound": {
                    "currency": "CurrencyA",                
                    "rate": {{Курс данной валюты по отношению ко второй валютной паре. Как правило один из курсов будет равен 1, а второй будет дробным числом}},
                },
                "outbound: {
                    "currency": "CurrencyB",                
                    "rate": {{Курс данной валюты по отношению ко второй валютной паре. Как правило один из курсов будет равен 1, а второй будет дробным числом}},
                }                
                "rate": {{Общий курс указанных валютных пар}},
                "min_amount": {{Минимальное количество первой валюты, доступное к процессингу}},
                "max_amount": "{{Максимальное количество первой валюты, доступное к процессингу}}",
                "reserve": {{Доступный размер резерва валюты выплаты}}
            },
            ...
        },
        "invalid": {
            "{{CurrencyX}}:{{CurrencyY}}": "Direction is not exists",
            ...
        }
    }
}
```


---
## Webhook уведомления о статусах заказов 📢
Вы можете (**а мы настоятельно рекомендуем**) подписать URL на вашем сервере на уведомления о изменении статусов заказов.

"***{{URL обратного вызова}}***" указывается в личном кабинете в вкладке "API настройки".
При изменении статуса заказа, система отправит HTTP запрос методом "***POST***" на ваш "***{{URL обратного вызова}}***" и телом запроса с объектом **{{[Order](#схема-объекта-order-используется-во-множестве-api-запросов-и-ответов)}}**. 

В ответ ваш сервер должен ответить **2XX** кодом. Если этого не произошло, webhook будет повторён с задержкой. При повторной неудаче задержка увеличится и webhook повторится. На момент написания документации у нас установлено 10 повторов уведомлений, однако это может измениться без уведомлений пользователей.

💡 Для тестирования вебхуков вы можете воспользоваться замечательным сторонним инструментом: [webhook.site](https://webhook.site), он позволяет получить URL для уведомлений и следить за их приходом посредством браузера.


### Проверка подлинности Webhook уведомлений

В целях безопасности (для проверки того, что webhook был прислан официальным источником), в webhook уведомлениях присутствует подпись, которая хеширует тело запроса с помощью ключа подписи.

Вы можете (а мы настоятельно рекомендуем) проверять подпись на вашей стороне, путем сравнения полученной из Header (X-Signature) подписи и подписью, сгенерированной на вашей стороне.

Подпись генерируется автоматически, абсолютно идентичным алгоритмом как в [подписи API запросов](#подпись-api-запросов)

---
