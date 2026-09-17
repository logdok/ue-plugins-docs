*[🇬🇧 English](../en/08-JetStream-Publisher.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 8. JetStream: публікація

Публікація в JetStream відрізняється від звичайного `Publish` ([4. Основний обмін
повідомленнями](05-Core-Messaging.md#публікація)) головним чином тим, що **сервер
підтверджує запис**: виклик не просто ставить повідомлення в чергу на відправку, а чекає,
поки сервер підтвердить, що повідомлення справді збережено в стрімі, і поверне номер
послідовності.

---

## Publish Message

```
Get JetStream
   │
   └─► Get Publisher
          │
          └─► Publish Message
                 Subject         : "orders.created"
                 Data            : "{\"id\":1,\"total\":99.99}"
                 Timeout Seconds : 5.0
                 │
                 └─ bSuccess → PubAck
                                  ├─ Stream     : "ORDERS"
                                  ├─ Sequence   : 1
                                  └─ Duplicate  : false
```

```cpp
UNatsJetStreamPublisherImpl* Publisher = Nats->GetJetStream()->Publisher();

Publisher->Publish(TEXT("orders.created"), TEXT("{\"id\":1,\"total\":99.99}"),
    [](TJetStreamResult<FJetStreamPubAck> Result)
    {
        if (Result.IsSuccess())
        {
            UE_LOG(LogTemp, Log, TEXT("Stream=%s Seq=%d"), *Result.Value.Stream, Result.Value.Sequence);
        }
        else
        {
            UE_LOG(LogTemp, Error, TEXT("Публікація не вдалася: %s"), *Result.Error.Message);
        }
    });
```

Якщо жоден стрім не захоплює `Subject` — операція завершується помилкою (сервер не відповідає,
результат приходить з `EJetStreamErrorCode::ServerError`), на відміну від звичайного `Publish`,
який у такому разі просто нічого не робить. Це очікувано: JetStream гарантує **збереження**, тож публікація в нікуди — це
помилка конфігурації, а не нормальний сценарій fire-and-forget.

### FJetStreamPubAck

| Поле | Опис |
|---|---|
| `Stream` | Який стрім зберіг повідомлення |
| `Sequence` | Номер послідовності, присвоєний повідомленню в стрімі |
| `Duplicate` | `true`, якщо повідомлення визнано дублікатом (див. [Message ID](#дедуплікація-publish-with-message-id) нижче) |
| `Domain` | Домен JetStream у мульти-тенантних розгортаннях (зазвичай порожньо) |

---

## Дедуплікація: Publish With Message ID

Мережа ненадійна: клієнт може не отримати підтвердження навіть тоді, коли сервер повідомлення
вже зберіг, — і повторити публікацію «про всяк випадок». Щоб таке повторення не створило
дублікат замовлення, укажіть власний унікальний ідентифікатор повідомлення:

```
Publish With Message ID
   Subject     : "orders.created"
   Data        : "{\"id\":1,\"total\":99.99}"
   Message Id  : "order-1-attempt"
```

```cpp
Publisher->PublishWithMsgId(TEXT("orders.created"), Data, TEXT("order-1-attempt"),
    [](TJetStreamResult<FJetStreamPubAck> Result)
    {
        if (Result.IsSuccess() && Result.Value.Duplicate)
        {
            // Сервер уже бачив цей Message Id — нове повідомлення НЕ додано,
            // Sequence вказує на вже наявний запис.
        }
    });
```

Якщо повідомлення з таким самим `Message Id` уже надходило протягом вікна дедуплікації
сервера (типово 2 хвилини), сервер поверне `Duplicate = true` і **не** створить другий
запис — `Sequence` у відповіді вказуватиме на вже наявне повідомлення. Це робить повторний
виклик безпечним: код може просто повторювати публікацію при невпевненості в результаті, не
боячись подвоєння даних.

---

## Оптимістична конкурентність: Publish With Expected Sequence

Коли кілька видавців можуть писати в один subject одночасно, а порядок важливий —
`Publish With Expected Sequence` дає публікацію типу «записати, лише якщо я знаю останній
стан стріму»:

```
Publish With Expected Sequence
   Subject           : "orders.created"
   Data              : (нове замовлення)
   Expected Last Seq : 41      ← я думаю, що останнє повідомлення в стрімі — #41
```

```cpp
Publisher->PublishWithExpectedSeq(Subject, Data, 41,
    [](TJetStreamResult<FJetStreamPubAck> Result)
    {
        if (!Result.IsSuccess())
        {
            // Хтось інший опублікував повідомлення між прочитанням стану й цим викликом —
            // ExpectedLastSeq більше не збігається. Прочитайте актуальний LastSeq
            // (Get Stream Info) і вирішіть, повторювати спробу чи ні.
        }
    });
```

Типовий цикл: прочитати `Get Stream Info` → `State.LastSeq`, спробувати публікацію з цим
значенням; при невдачі — прочитати актуальний `LastSeq` заново й вирішити, чи повторювати.

---

## Bytes-варіанти публікації

Кожен із чотирьох методів вище має двійковий відповідник — той самий принцип, що й у
[6. Бінарні дані](06-Binary-Data.md): байти передаються точно, без текстового кодування.

| Текст | Байти |
|---|---|
| `Publish Message` | `Publish Bytes` |
| `Publish With Headers` | `Publish Bytes With Headers` |
| `Publish With Message ID` | `Publish Bytes With Message ID` |
| `Publish With Expected Sequence` | `Publish Bytes With Expected Sequence` |

```
Publish Bytes
   Subject : "assets.uploaded"
   Payload : (масив байтів, наприклад стиснений файл)
```

```cpp
Publisher->PublishBytes(TEXT("assets.uploaded"), CompressedBytes, Callback);
```

---

## Публікація з заголовками

```
Make Map (String → String)
   ["Content-Type"] = "application/json"
   │
   └─► Publish With Headers
          Subject : "orders.created"
          Data    : "{\"id\":1}"
          Headers : (мапа зверху)
```

Заголовки в JetStream працюють так само, як у Core
([5. Основний обмін повідомленнями](05-Core-Messaging.md#publish-with-headers)) — це
звичайні заголовки NATS-повідомлення, JetStream їх просто зберігає разом із даними.

---

## Подія On Publish Ack

Окрім результату, що повертається колбеком кожного окремого виклику, є загальна подія —
зручна для централізованого моніторингу всіх публікацій, а не обробки кожної окремо:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Bind Event to On JetStream Pub Ack
            │
            └─► (bSuccess, PubAck) → оновити лічильник в UI, залогувати
```

Спрацьовує для **кожної** успішної й неуспішної публікації через Publisher, незалежно від
того, звідки саме вона була викликана.

---

## Практичний приклад: надійна відправка замовлення з повтором

```cpp
void AOrderService::SubmitOrder(const FString& OrderId, const FString& OrderJson)
{
    UNatsJetStreamPublisherImpl* Publisher = Nats->GetJetStream()->Publisher();

    // OrderId як Message Id: повторний виклик (наприклад, після тайм-ауту мережі)
    // безпечний — дубліката не буде.
    Publisher->PublishWithMsgId(TEXT("orders.created"), OrderJson, OrderId,
        [this, OrderId](TJetStreamResult<FJetStreamPubAck> Result)
        {
            if (Result.IsSuccess())
            {
                UE_LOG(LogTemp, Log, TEXT("Замовлення %s: seq=%d, дублікат=%s"),
                    *OrderId, Result.Value.Sequence, Result.Value.Duplicate ? TEXT("так") : TEXT("ні"));
            }
            else if (Result.Error.Code == EJetStreamErrorCode::Timeout)
            {
                // Мережева проблема — той самий Message Id безпечно повторити.
                RetrySubmitOrder(OrderId);
            }
        });
}
```

---

**Далі:** [9. JetStream: споживачі](09-JetStream-Consumers.md)
