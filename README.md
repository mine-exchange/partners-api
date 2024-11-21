# Партнёрское API

## Типичные сценарии работы 🔀

### Работа с заказами:
- [Создать заказ](#создание-заказов);
- Проверка статуса, обычно [по webhook уведомлению](#webhook-уведомления-о-статусах-заказов-), однако в некоторых случаях можно сделать явный запрос для [получения состояния группы заказов](#получение-заказов);
- Иногда требуется [удалить заказ](#отмена-заказов), пока он не пошёл в работу.

### Иногда требуется уточнить:
- [Пары, курсы, ограничения, комиссии](#получение-текущих-пар-курсов-ограничений-минимальноймаксимальной-суммы-и-резервов)
- [Балансы MineCoin счетов](#получение-балансов-minecoin)
---

## ⚡️ Быстрый старт (коллекция методов с автоматизацией подписи в Postman)

Для вашего удобства мы создали коллекцию в [postman](https://www.postman.com/), которая повторяет все методы API. Вы можете протестировать на ней функциональность и поиграть с запросами.
1. В postman выбрать пункт меню "Import file", например во вкладке "Home"
2. Перейти во вкладку "Link" и импортировать следующую ссылку: `https://api.postman.com/collections/6328473-21626561-38fb-44cd-8c82-e24bc4047d2a?access_key=PMAT-01HT2VGXW98N3BSTC169WZ0HC7`
5. Перейти курсором на импортированную коллекцию, выбрать вкладку "Variables" и вписать вашу конфигурацию в колонке "Current value"
## ⚠️ Финансовый контроль
На некоторых парах действует пересчёт заказов. Это значит что в случае поступления неверной суммы от клиента, заказ пересчитывается и выполняется исходя из суммы поступления. В момент получения финального статуса заказа (предпочтительно по webhook), необходимо учитывать суммы в ответе API и учитывать именно их в вашей системе. Недопустимо доверять зачислению без сверки сумм.

Также, рекомендуем регулярно сверять балансы от нашего API и вашей системы.


## ⚠️ Контроль сетевых потерь
Связь между серверами хоть и редко, но может обрываться, поэтому необходимо валидировать, являются ли полученные в ответ данные ожидаемыми вами:
1. Если запрос прошёл корректно, клиент получает 2xx http код. В иных случаях произошла ошибка запроса, а [в теле ответа содержатся её подробности](#стандартный-ответ-об-ошибке-для-всех-запросов). Альтернативный вариант проверки прохождения запроса — проверка параметра success в json объекте тела ответа (1 если запрос прошёл, иначе 0).
2. API всегда отвечает JSON ответом. Если вы не получили корректный JSON — вы не можете точно знать, прошёл ли ваш запрос или был потерян по пути. В случае критичных запросов (например, выплата), следует перепроверять такие запросы любым доступным вам способом и не считать запрос выполненным или не выполненным, пока не получите точного подтверждения произошедшего.
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
    "success": 0,
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
    "inbound": {
        "currency": "{{Валюта отправки}}",
        "account": "{{Аккаунт отправителя}}",
        "amount": "{{Сумма отправки}}",
        "amount_initial": "{{Изначальная сумма отправки; Используется в API ответах}}",
        "txid": "{{ID входящей транзакции; Используется в API ответах}}", //nullable
        "code": "{{Код для оплаты; Используется в API запросах}}" //nullable        
    },
    "outbound": {
        "currency": "{{Валюта выплаты}}",
        "account": "{{Аккаунт получателя}}",
        "amount": "{{Сумма выплаты}}",
        "amount_initial": "{{Изначальная сумма выплаты; Используется в API ответах}}", 
        "txid": "{{ID входящей транзакции; Используется в API ответах}}", //nullable
        "memo": "{{Memo криптовалютного кошелька; Используется в API запросах}}" //nullable       
    },
    "is_customer_confirmation_need": "{{Флаг, показывающий необходимо ли подтверждать оплату по заказу; Используется в API ответах}}",
    "rate": "{{Курс обмена; Используется в API ответах}}",
    "rate_initial": "{{Изначальный курс обмена; Используется в API ответах}}",
    "rate_type": "{{Тип курса обмена: inbound|outbound; Используется в API ответах}}",
    "comment": "{{Комментарий к заказу (необязателен)}}",
    "status": "{{Статус операции: 'new|processing|incoming_unconfirmed|incoming_confirmed|moderation|error|canceled|success'; Используется в API ответах}}",
    "created_at": "{{Время создания в формате ISO 8601; Используется в API ответах}}",
    "updated_at": "{{Время изменения в формате ISO 8601; Используется в API ответах}}"
}
```

### Что такое amount_initial?

При [инициации заказа](#создание-заказов) amount_initial не передаётся. Указывать требуется outbound.amount или inbound.amount, который сразу зафиксируется в amount_initial

Причины почему amount и amount_initial могут отличаться:
- Был создан заказ CARDUAH -> BTC, на 0.1 BTC, в результате рассчёта получилось, что нужно оплатить 100к грн, но было оплачено только 50к, движок пересчитал заказ и вылатил 0.05 BTC
- Иногда могут поплыть копейки из за подкапотных пересчётов.

### Возможные значения параметра status

| **status** | **Описание** |
|--|--|
| **new** | Новый заказ, помещён в очередь, не обработан. |
| **processing** | Заказ помещён в обработку, ожидает действий от пользователя. |
| **incoming_unconfirmed** | В некоторых случаях после оплаты клиентом система видит транзакцию в сети, однако средства ещё не поступили и попытка может провалиться. В этом случае ставится этот статус, который подразумевает ожидание входящего платежа. |
| **incoming_confirmed** | Платёж от клиента подтверждён. |
| **moderation** | Заказ проходит модерацию менеджера сервиса. Редкий статус. |
| ❓ **error** | Ошибка в заказе, статус не финальный, свяжитесь с техподдержкой для уточнения ситуации. |
| 🏁 **canceled** | Заказ отменён (пользователем или системой). В парах c MineCoin данный статус означает, что заказ получил финальный статус, деньги по заказу не отправлены и возвращены на счет MineCoin. Возможные причины: ошибка в реквизитах получателя, ограничения банка эмитента, иные причины. После получения этого статуса можно повторять отправку. |
| 🏁 **success** | Выплата средств произведена, заказ выплачен. |


🏁 — Финальные статусы.

❓ **error** — Произошла неизвестная исключительная ситуация, требуется связаться с поддержкой для уточнения деталей. Повторную отправку делать нельзя.

---


### Обязательные параметры, дополняющие информацию о заказах:

#### Order.extra — Дополнительные данные, которые партнёр мог передать при создании заказа (используется в Запросах и Ответах)

```js
{
    "extra": {
        "success_url": "{{Адрес для перенаправления после успешной оплаты}}", //nullable
        "failed_url": "{{Адрес для перенаправления после неуспешной оплаты}}", //nullable
        "client": {
            "id": "{{ID пользователя в вашей системе}}",
            "email": "{{Email адрес пользователя вашей системы}}",
            "phone": "{{Номер телефона пользователя в вашей системы}}", //nullable
            "first_name": "{{Имя пользователя вашей системы}}", //nullable
            "last_name": "{{Фамилия пользователя вашей системы}}", //nullable
            "middle_nanme": "{{Отчество пользователя вашей системы}}", //nullable
            "ip": "{{IP-адрес пользователя вашей системы, который совершает действие}}",
            "user_agent": "{{Useragent (браузер) пользователя. Примечание для php разработчика: содержится в $_SERVER['HTTP_USER_AGENT']}}"
        }
    }
}
```

### Необязательные, но возможные параметры, дополняющие информацию о заказах:

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
            "memo": null|"{{memo}}",
            "error": null|"{{Описание ошибки}}"
        }
    }
}
```   

**Реквизиты payment_variants в зависимости от приёмной платёжной системы**

⚠️ Данное описание реквизитов актуально на момент написания и служит исключительно для ориентира интегратора. Набор параметров может измениться.

| **Платёжная система** | **Объект в payment_variants** | **Описание** |
|:--|:--|:--|
| **Jumbo**                                 | invoice | (url) Ссылка для переадресации клиента на выставленный счёт |
| **Payeer**                                | direct  | (url) Ссылка для переадресации клиента на страницу прямой оплаты на кошелёк |
| **BITCOIN** (BTC)                         | manual  | (number) Номер btc кошелька для оплаты |
| **Остальные мерчанты**                    | direct  | (url) Ссылка на платёжную форму мерчанта |

**Каждый элемент объекта "payment_variants" содержит следующие параметры:**
- error (Обязательный параметр. По умолчанию null, нет ошибки. В противном случае текстовое описание ошибки, из за которой невозможно использование данного метода оплаты).
- Один из следующих реквизитов оплаты, обязательный параметр:
    - url: адрес платёжной системы, куда необходимо переадресовать клиента.
    - number: реквизиты, по которым клиенту необходимо вручную перевести оплату.
- note (Текстовое примечание для упрощения интеграции программистам и упрощения процесса интеграции. Необязательный параметр, иногда присутствующий).



#### Order.details — Используется в Ответах для дополнения заказа подробностями

На данный момент для выплат с некоторых платёжных систем используется генерация чеков в Order.details.receipts. Клиенты могут загрузить чеки и подтвердить факт выплаты.
```js
{
    "details": {
        "comment_for_client": "{{comment_for_client}}",
        "payout_receipts": {
            "{{payment_system_transaction_id}}": {
                "pdf": "{{https://************.pdf}}",
                "jpeg": "{{https://************.jpeg}}",
                "amount": {{amount}}
            },
            ...
        },
        ... 
    }
}
```        


---
## Методы API 🔢

### Получение заказов

⚠️ Мы не рекомендуем регулярно пользоваться данным методом во избежание задержек. Для уведомлений о изменениях состояния заказов реализован инструмент "[Webhook уведомления о заказах](#webhook-уведомления-о-статусах-заказов-)"

**[GET] /{{API_PARTNERS_PREFIX}}/orders?ids={{id1}},{{id2}},...&page={{pages_count}}&per_page={{items_count}}**
- ids: перечисленные через запятую номера заказов партнёра. Допустимо указать один конкретный заказ;
- page: необязательный параметр, страница списка, которую требуется получить;
- per_page: количество элементов на 1 страницу;

#### Успешный ответ (200)
```js
{
    "success": 1,
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

Данный запрос создаёт заказы, после создания заказа, для его выполнения необходимо [совершить оплату по реквизитам](#orderpayment_variants--возможные-способы-оплаты-заказа-клиентом-используется-в-ответах), выданным в ответе.

**[POST] /{{API_PARTNERS_PREFIX}}/orders**
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
4. Обязательно требуется добавление [extra параметров](#orderextra--дополнительные-данные-которые-партнёр-мог-передать-при-создании-заказа-используется-в-запросах-и-ответах) с данными о заказе и пользователе в вашей системе.
5. Вы можете указать поле "comment" для вашего удобства, однако получатель его не увидит. Данное поле будет отображаться в информации о заказе по API и в личном кабинете.

💡 Если у вас особый кейс, в котором не обязательно, чтобы ID заказа соответствовал ID в вашей системе, API может сгенерировать {{order_id}} заказа случайным образом. Для этого укажите его следующим образом:
```js
{
    "order_id": "auto-uid",
}
```

#### Успешный ответ (200)
```js
{
    "success": 1,
    "code": 200,
    "data": {
        "invalid": {
            "{{ID заказа партнера}}": "{{Сообщение об ошибке}}",
            ...
        },
        "created": {
            "{{ID заказа партнера}}": {{Order}},
            ...
        }
    }
}
```

---
### Подтверждение оплаты заказов

ℹ️ Допустимо подтверждение заказов только в статусе "new". Подтверждение необходимо, если в объекте [`Order`](#схема-объекта--order---используется-во-множестве-api-запросов-и-ответов-) флаг `is_customer_confirmation_need` установлен в значение `true`.

**[PUT] /{{API_PARTNERS_PREFIX}}/orders/confirm-by-customer**

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
    "success": 1,
    "code": 200,
    "data": {
        "invalid": {
            "{{ID заказа партнера}}": "{{Текстовое представление причины, по которой не удалось выполнить подтверждение}}",
            ...
        },
        "confirmed": {
            "{{ID заказа партнера}}": {{Order}}
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

**[DELETE] /{{API_PARTNERS_PREFIX}}/orders**
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
    "success": 1,
    "code": 200,
    "data": {
        "invalid": {
            "{{ID заказа партнера}}": "{{Сообщение об ошибке}}",
            ...
        },
        "сanceled": {
            "{{ID заказа партнера}}": {{Order}},        
            ...
        }
    }
}
```
ℹ️ Смотрите также: [Описание cхемы объекта "Order"](#схема-объекта-order-используется-во-множестве-api-запросов-и-ответов).


---
### Получение балансов MineCoin
Балансы депозитного счёта в нашей системе (каждый счёт привязан к определённой валюте).

**[GET] /{{API_PARTNERS_PREFIX}}/balances**

#### Успешный ответ (200)
```js
{
    "success": 1,
    "code": 200,
    "data": {
        "BTC": {{Количество средств на балансе}},
        "USDT": {{Количество средств на балансе}},
        "UAH": {{Количество средств на балансе}}        
    }
}
```

---
### Получение текущих пар, курсов, ограничений минимальной/максимальной суммы и резервов

**[GET] /{{API_PARTNERS_PREFIX}}/pairs?names={{CurrencyA}}:{{CurrencyB}},{{CurrencyX}}:{{CurrencyY}},...&inbound_currencies={{CurrencyA}},{{CurrencyB}}...&outbound_currencies={{CurrencyA}},{{CurrencyB}}**
- names: необязательный параметр, перечисленные через запятую названия пар в формате: "{{CurrencyA}}:{{CurrencyB}}";
- inbound_currencies: необязательный параметр, перечисленные через запятую входящие коды валют;
- outbound_currencies: необязательный параметр, перечисленные через запятую исходящие коды валют;

#### Успешный ответ (200)
```js
{
    "success": 1,
    "code": 200,
    "data": {
        "{{CurrencyA}}:{{CurrencyB}}": {
            "inbound": {
                "currency": "CurrencyA",
                "rate": {{Курс данной валюты по отношению ко второй валютной паре. Как правило один из курсов будет равен 1, а второй будет дробным числом}},
                "is_account_required": {{Необходимо ли передавать account поле при создании заказа}}
            },
            "outbound": {
                "currency": "CurrencyB",
                "rate": {{Курс данной валюты по отношению ко второй валютной паре. Как правило один из курсов будет равен 1, а второй будет дробным числом}},
                "is_account_required": {{Необходимо ли передавать account поле при создании заказа}}
            },
            "rate": {{Общий курс указанных валютных пар}},
            "min_amount": {{Минимальное количество первой валюты, доступное к процессингу}},
            "max_amount": "{{Максимальное количество первой валюты, доступное к процессингу}}",
            "reserve": {{Доступный размер резерва валюты выплаты}}
        },
        ...
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
