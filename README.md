<div align="center">

# 💳 Платёжные интеграции в Узбекистане

**Payme · Click · Uzum** — три разные интеграции за одним интерфейсом

[![PHP](https://img.shields.io/badge/PHP-8.3-777BB4?style=for-the-badge&logo=php&logoColor=white)](https://php.net)
![Провайдеров](https://img.shields.io/badge/провайдеров-3-6366f1?style=flat-square)
![Идемпотентность](https://img.shields.io/badge/идемпотентность-обязательна-22c55e?style=flat-square)

</div>

---

## 🎭 Главное заблуждение

«Подключить оплату» звучит как одна задача. На деле это **три разные интеграции**:
у каждого провайдера свой формат callback, своя схема подписи, свой набор
состояний платежа и своя логика подтверждения.

Общего между ними ровно столько, сколько можно спрятать за один интерфейс:

```php
interface PaymentGateway
{
    public function providerName(): string;

    /** Ссылка, на которую браузер покупателя уходит платить. */
    public function buildCheckoutUrl(array $order, int $paymentTransactionId): string;

    /** Обработка одного входящего webhook этого провайдера. */
    public function handleWebhook(string $rawBody, array $headers): array;
}
```

Три метода. Всё остальное — различия, и попытка их обобщить даёт абстракцию,
которая течёт при первом же изменении на стороне провайдера.

---

## 🔄 Жизненный цикл платежа

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'ui-sans-serif,-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica,Arial,sans-serif','fontSize':'15px','textColor':'#1e1b4b','lineColor':'#4338ca','primaryColor':'#e0e7ff','primaryTextColor':'#1e1b4b','primaryBorderColor':'#4338ca','secondaryColor':'#ede9fe','tertiaryColor':'#f5f3ff','mainBkg':'#c7d2fe','nodeBorder':'#4338ca','nodeTextColor':'#1e1b4b','edgeLabelBackground':'#e0e7ff','attributeBackgroundColorOdd':'#eef2ff','attributeBackgroundColorEven':'#e0e7ff','noteBkgColor':'#fef3c7','noteTextColor':'#1a1a1a','noteBorderColor':'#b45309','clusterBkg':'#f5f3ff','clusterBorder':'#4338ca','labelBoxBkgColor':'#e0e7ff','labelBoxBorderColor':'#4338ca','labelTextColor':'#1e1b4b','actorBkg':'#c7d2fe','actorBorder':'#4338ca','actorTextColor':'#1e1b4b','actorLineColor':'#4338ca','signalColor':'#4338ca','signalTextColor':'#1e1b4b','sequenceNumberColor':'#ffffff','activationBkgColor':'#ddd6fe','activationBorderColor':'#4338ca','transitionColor':'#4338ca','transitionLabelColor':'#1e1b4b','stateBkg':'#c7d2fe','stateLabelColor':'#1e1b4b','altBackground':'#f5f3ff','compositeBackground':'#f5f3ff','compositeBorder':'#4338ca','compositeTitleBackground':'#e0e7ff','specialStateColor':'#4338ca','innerEndBackground':'#4338ca'}}}%%
stateDiagram-v2
    [*] --> created: создана транзакция
    created --> pending: покупатель ушёл на оплату
    pending --> held: средства захолдированы
    pending --> failed: отказ банка
    held --> paid: подтверждено
    held --> cancelled: отменено до списания
    paid --> refunded: возврат
    failed --> [*]
    cancelled --> [*]
    refunded --> [*]
    paid --> [*]

    note right of held
        Двухфазная схема есть не у всех.
        Там, где её нет, pending переходит
        сразу в paid — и код должен это
        выдерживать без отдельной ветки.
    end note
```

Состояние движется **только вперёд**. Переход `paid → pending` невозможен ни при
каких входящих данных: повторный или опоздавший webhook не должен откатывать уже
подтверждённый платёж. Это правило проще всего нарушить именно тем, что обработчик
просто присваивает пришедший статус.

---

## 🔒 Идемпотентность: главное требование, а не деталь

Провайдер повторит webhook, если не получил ответ вовремя. Сеть моргнула, сервер
перезапустился, обработчик отработал 31 секунду вместо 30 — придёт ещё раз.

Без защиты это означает **двойное зачисление**. В учебном центре — оплаченный
дважды месяц, в магазине — два заказа, в кассе — расхождение, которое придётся
разбирать вручную по выписке.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'ui-sans-serif,-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica,Arial,sans-serif','fontSize':'15px','textColor':'#1e1b4b','lineColor':'#4338ca','primaryColor':'#e0e7ff','primaryTextColor':'#1e1b4b','primaryBorderColor':'#4338ca','secondaryColor':'#ede9fe','tertiaryColor':'#f5f3ff','mainBkg':'#c7d2fe','nodeBorder':'#4338ca','nodeTextColor':'#1e1b4b','edgeLabelBackground':'#e0e7ff','attributeBackgroundColorOdd':'#eef2ff','attributeBackgroundColorEven':'#e0e7ff','noteBkgColor':'#fef3c7','noteTextColor':'#1a1a1a','noteBorderColor':'#b45309','clusterBkg':'#f5f3ff','clusterBorder':'#4338ca','labelBoxBkgColor':'#e0e7ff','labelBoxBorderColor':'#4338ca','labelTextColor':'#1e1b4b','actorBkg':'#c7d2fe','actorBorder':'#4338ca','actorTextColor':'#1e1b4b','actorLineColor':'#4338ca','signalColor':'#4338ca','signalTextColor':'#1e1b4b','sequenceNumberColor':'#ffffff','activationBkgColor':'#ddd6fe','activationBorderColor':'#4338ca','transitionColor':'#4338ca','transitionLabelColor':'#1e1b4b','stateBkg':'#c7d2fe','stateLabelColor':'#1e1b4b','altBackground':'#f5f3ff','compositeBackground':'#f5f3ff','compositeBorder':'#4338ca','compositeTitleBackground':'#e0e7ff','specialStateColor':'#4338ca','innerEndBackground':'#4338ca'}}}%%
sequenceDiagram
    participant P as Платёжная система
    participant W as Webhook-обработчик
    participant DB as База

    P->>W: callback (transaction_id=A, paid)
    W->>W: проверка подписи
    W->>DB: SELECT ... WHERE provider_tx_id = A FOR UPDATE
    DB-->>W: не найдено
    W->>DB: INSERT транзакция, статус paid
    W->>DB: начислить заказу
    W-->>P: 200 OK

    Note over P,W: ответ потерялся в сети

    P->>W: тот же callback (transaction_id=A)
    W->>W: проверка подписи
    W->>DB: SELECT ... WHERE provider_tx_id = A FOR UPDATE
    DB-->>W: найдено, уже paid
    W-->>P: 200 OK, повторного начисления нет
```

Три вещи, без которых это не работает:

1. **Идентификатор провайдера уникален в базе.** `UNIQUE (provider, provider_tx_id)` —
   на уровне схемы, а не проверкой в коде. Код проверит и проиграет гонку, индекс — нет.
2. **Блокировка строки на время обработки.** Два одновременных повтора иначе оба
   пройдут проверку «не найдено».
3. **Ответ 200 на повтор.** Если отвечать ошибкой, провайдер будет слать снова и
   снова, пока не упрётся в лимит и не пометит платёж проблемным.

---

## 🛡️ Подпись проверяется до всего остального

```php
public function handleWebhook(string $rawBody, array $headers): array
{
    // Подпись — первое, что делает обработчик. Не после разбора JSON,
    // не после поиска заказа: до любой работы с данными из тела запроса.
    if (!$this->signatureIsValid($rawBody, $headers)) {
        return ['code' => self::ERROR_UNAUTHORIZED];
    }

    $payload = json_decode($rawBody, true);
    // ... дальше бизнес-логика
}
```

Разбирать JSON до проверки подписи — значит обрабатывать данные, которые прислал
кто угодно. В платёжном контуре это неприемлемо.

---

## 💵 Деньги — целые числа, всегда

Суммы передаются и хранятся **в минимальных единицах** (тийинах), а не в сумах с
дробью. `float` в денежном расчёте даёт копеечные расхождения, которые всплывают
на сверке за месяц и ищутся часами.

В базе — `DECIMAL`, в API провайдера — целое. Конверсия ровно в одном месте, на
границе с провайдером, а не по всему коду.

---

## ⏳ Что не автоматизируется и занимает больше времени, чем код

Готовая интеграция — не главная часть работы. Больше всего времени уходит на:

- **Мерчант-онбординг.** Договор, документы, проверка юрлица. По рынку — около
  двух недель от подачи, и ускорить это технически невозможно.
- **Получение доступа к песочнице** и согласование URL webhook.
- **Сверку.** Отчёт «платежи провайдера против заказов в системе» за период —
  первое, что спросит бухгалтер, и то, чего почти никогда нет в коробочных решениях.
- **Возвраты.** Сценарий, про который вспоминают после запуска, а он меняет схему
  данных.

---

## 🧾 Фискализация: оплата прошла — это ещё не всё

Онлайн-оплата в Узбекистане требует фискального чека. Платёжная система может
фискализировать его за вас, выступая **комиссионером**, но только если продавец
сам зарегистрировал её в личном кабинете `my.soliq.uz`.

Не зарегистрировал — чеки не уходят, и узнаётся это обычно поздно. Подробнее:
[uz-fiscal-compliance-notes](https://github.com/Shohruh1997/uz-fiscal-compliance-notes).

---

## ✅ Чек-лист перед запуском

- [ ] Подпись проверяется до разбора тела запроса
- [ ] `UNIQUE (provider, provider_tx_id)` в схеме
- [ ] Повторный webhook возвращает 200 и не начисляет второй раз
- [ ] Статус движется только вперёд
- [ ] Суммы целые, в минимальных единицах
- [ ] Неизвестный статус логируется, а не роняет обработчик
- [ ] Есть отчёт сверки за период
- [ ] Продуман сценарий возврата
- [ ] Комиссионер зарегистрирован в `my.soliq.uz`, чеки видны в кабинете

---

## 🔗 Смежные заметки

- [webhook-idempotency](https://github.com/Shohruh1997/webhook-idempotency-php) — рабочая PHP-реализация идемпотентности из этой заметки, MIT
- [db-schema-notes](https://github.com/Shohruh1997/db-schema-notes) — схемы БД, в том числе таблицы транзакций
- [uz-fiscal-compliance-notes](https://github.com/Shohruh1997/uz-fiscal-compliance-notes) — ИКПУ, касса, ЭСФ
- [php-layered-architecture-notes](https://github.com/Shohruh1997/php-layered-architecture-notes) — где в слоях живёт шлюз

---

<div align="center">

**Шохрух Рузиев** · backend-разработчик, Ташкент

[![Сайт](https://img.shields.io/badge/ecomdev.uz-6366f1?style=for-the-badge&logo=googlechrome&logoColor=white)](https://ecomdev.uz)
[![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/EcomDev_uz)

Исходный код систем — в приватных репозиториях, доступ по запросу.

</div>
