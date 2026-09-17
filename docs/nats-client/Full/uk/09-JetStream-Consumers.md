*[🇬🇧 English](../en/09-JetStream-Consumers.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 9. JetStream: споживачі

Стрім зберігає повідомлення; споживач (Consumer) — це курсор, яким їх звідти читають.
Концептуально розібрано в [1. Вступ до NATS](01-Introduction.md#споживач-consumer-курсор-читання-зі-стріму).
Цей розділ — практичний: як створити споживача, обрати push чи pull, і як підтверджувати
опрацьовані повідомлення.

---

## Push проти Pull: що обрати

| | **Push-споживач** | **Pull-споживач** |
|---|---|---|
| Хто ініціює доставку | Сервер сам надсилає, щойно з'являється повідомлення | Ви самі просите: «дай до N повідомлень» |
| Швидкість реакції | Миттєва | Залежить від того, як часто ви запитуєте |
| Контроль темпу обробки | Обмежений | Повний — просите рівно стільки, скільки готові обробити |
| Типове застосування | Подієва обробка в реальному часі: сповіщення, live-оновлення стану | Пакетна обробка, черги завдань, воркери, що обробляють по кілька елементів за раз |

Вибір визначається одним полем конфігурації — `DeliverSubject`: заповнене — push-споживач,
порожнє — pull.

---

## Конфігурація споживача

`FJetStreamConsumerConfig`:

| Поле | Тип | За замовчуванням | Опис |
|---|---|---|---|
| `Name` | `FString` | — | Ім'я споживача, наприклад `"ORDERS_PROCESSOR"`. Порожнє ім'я створює ефемерного (тимчасового) споживача |
| `DeliverPolicy` | `EJetStreamDeliverPolicy` | `All` | Звідки почати читання — [нижче](#deliverpolicy-звідки-почати-читання) |
| `AckPolicy` | `EJetStreamAckPolicy` | `Explicit` | Як підтверджувати — [нижче](#ackpolicy-як-підтверджувати) |
| `AckWait` | `float` | `30.0` | Скільки чекати підтвердження (секунди), перш ніж надіслати повідомлення повторно |
| `MaxDeliver` | `int32` | `-1` (без ліміту) | Скільки разів намагатися доставити одне повідомлення |
| `ReplayPolicy` | `EJetStreamReplayPolicy` | `Instant` | `Instant` — якнайшвидше. `Original` — з тими самими інтервалами, що й при первинній публікації |
| `FilterSubject` | `FString` | порожньо | Отримувати лише повідомлення з цим subject (для стрімів із кількома subject'ами) |
| `OptStartSeq` | `int32` | `0` | Початкова послідовність (лише для `DeliverPolicy = ByStartSequence`) |
| `OptStartTime` | `int32` | `0` | Початковий час, Unix timestamp (лише для `DeliverPolicy = ByStartTime`) |
| `DeliverSubject` | `FString` | порожньо | Заповнено → push-споживач. Порожньо → pull-споживач |

### DeliverPolicy: звідки почати читання

| Значення | Поведінка |
|---|---|
| `All` | Від найпершого повідомлення в стрімі — повне відтворення історії |
| `New` | Лише повідомлення, опубліковані **після** створення споживача |
| `Last` | Почати з останнього наявного повідомлення |
| `LastPerSubject` | Останнє повідомлення для **кожного** subject у стрімі з кількома subject'ами — зручно для «поточного стану» |
| `ByStartSequence` | З конкретного номера послідовності (`OptStartSeq`) |
| `ByStartTime` | З конкретного моменту часу (`OptStartTime`) |

### AckPolicy: як підтверджувати

| Значення | Поведінка |
|---|---|
| `Explicit` (типово, рекомендовано) | Кожне повідомлення підтверджується окремо, у довільному порядку |
| `All` | Підтвердження повідомлення №N автоматично підтверджує й усі попередні — лише строго по порядку |
| `None` | Підтвердження не потрібне: повідомлення вважається доставленим одразу. Ризик втрати при збої обробника |

### Builder (C++)

```cpp
FJetStreamConsumerConfig Config = FJetStreamConsumerConfigBuilder()
    .WithName(TEXT("ORDERS_PROCESSOR"))
    .WithDeliverPolicy(EJetStreamDeliverPolicy::New)
    .WithAckPolicy(EJetStreamAckPolicy::Explicit)
    .WithAckWaitSeconds(60.0f)
    .WithFilterSubject(TEXT("orders.created"))
    .Build();
```

Для push-споживача — `AsPushConsumer()` (сам генерує унікальний `DeliverSubject`, якщо ви
не задали свій) або `WithDeliverSubject(TEXT("..."))` явно.

---

## Push-споживач

### Create Consumer + Subscribe To Consumer

```
Get Consumers Manager
   │
   ├─► Create Consumer
   │      Stream Name          : "ORDERS"
   │      Config → Name        : "ORDERS_LIVE"
   │      Config → Deliver Subject : "_INBOX.orders_live"   ← будь-який унікальний рядок
   │      │
   │      └─ bSuccess ──►
   │
   └─► Subscribe To Consumer
          Stream Name    : "ORDERS"
          Consumer Name  : "ORDERS_LIVE"
```

Після `Subscribe To Consumer` повідомлення надходять подією:

```
Get Consumers Manager
   │
   └─► Bind Event to On Message Received
            │
            └─► (Message: FJetStreamMessage)
                    │
                    ├─► Break Jet Stream Message
                    │       ├─ Nats Msg → Data     (сам вміст)
                    │       ├─ Sequence
                    │       └─ Num Delivered        (>1 означає повторну доставку)
                    │
                    └─► Ack Message (Message)   ← обов'язково після успішної обробки!
```

```cpp
UNatsConsumerManagerImpl* Consumers = Nats->GetJetStream()->Consumers();

FJetStreamConsumerConfig Config = FJetStreamConsumerConfigBuilder()
    .WithName(TEXT("ORDERS_LIVE"))
    .AsPushConsumer()
    .Build();

Consumers->CreateConsumer(TEXT("ORDERS"), Config,
    [Consumers](TJetStreamResult<FJetStreamConsumerInfo> Result)
    {
        if (Result.IsSuccess())
        {
            Consumers->Subscribe(TEXT("ORDERS"), Result.Value.Config.Name,
                [](TJetStreamResult<FJetStreamConsumerInfo>) {});
        }
    });

Consumers->OnMessageReceived.AddDynamic(this, &AMyActor::HandleOrder);

// ...
void AMyActor::HandleOrder(const FJetStreamMessage& Message)
{
    ProcessOrder(Message.NatsMsg.Data);
    Nats->GetJetStream()->Consumers()->AckMessage(Message);
}
```

> **Подія одна на всі push-споживачі.** Якщо підписані на кількох споживачів одночасно —
> розрізняйте повідомлення за `Message.Consumer` або `Message.NatsMsg.Subject`.

---

## Pull-споживач

Конфігурація без `DeliverSubject`, і замість підписки — явний запит потрібної кількості
повідомлень:

```
Get Consumers Manager
   │
   ├─► Create Consumer
   │      Stream Name    : "ORDERS"
   │      Config → Name  : "ORDERS_BATCH"
   │      (Deliver Subject лишається порожнім)
   │
   └─► Pull Messages
          Stream Name     : "ORDERS"
          Consumer Name   : "ORDERS_BATCH"
          Batch Size      : 10
          Timeout Seconds : 5.0
          │
          └─ bSuccess → Messages : (масив до 10 елементів)
                            │
                            └─► For Each Loop
                                    └─► Ack Message
```

```cpp
FJetStreamConsumerConfig Config = FJetStreamConsumerConfigBuilder()
    .WithName(TEXT("ORDERS_BATCH"))
    .Build(); // DeliverSubject порожній за замовчуванням — pull

Consumers->CreateConsumer(TEXT("ORDERS"), Config, [](auto) {});

Consumers->PullMessages(TEXT("ORDERS"), TEXT("ORDERS_BATCH"), 10, 5.0f,
    [Consumers](TJetStreamResult<TArray<FJetStreamMessage>> Result)
    {
        if (Result.IsSuccess())
        {
            for (const FJetStreamMessage& Message : Result.Value)
            {
                ProcessOrder(Message.NatsMsg.Data);
                Consumers->AckMessage(Message);
            }
        }
    });
```

Якщо за `Timeout Seconds` не назбиралося жодного повідомлення — `Messages` повертається
порожнім масивом з `bSuccess = true`: порожня черга — це нормальний результат, а не помилка.
Якщо повідомлень менше, ніж `Batch Size` — повертається стільки, скільки є.

---

## Підтвердження: Ack / Nak / Term

| Нода | Сенс | Наслідок |
|---|---|---|
| **Ack Message** | «Оброблено успішно» | Не буде доставлено повторно. Для `WorkQueue`-стріму — видаляється зі стріму |
| **Nak Message** | «Не вдалося, спробуй ще» | Повторна доставка після короткої затримки; `NumDelivered` зростає |
| **Term Message** | «Не намагайся більше» | Повідомлення НІКОЛИ не буде доставлено повторно, навіть без успішної обробки |

```
[після обробки замовлення]
   │
   ├─ успіх                     → Ack Message
   ├─ тимчасова помилка (БД недоступна) → Nak Message
   └─ дані пошкоджені/невалідні → Term Message
```

**Головне правило:** якщо `AckWait` минув, а підтвердження не було — сервер вважає
повідомлення необробленим і доставляє його знову. Це стосується і push-, і pull-споживачів
однаково. Якщо обробка систематично довша за `AckWait` — збільшуйте `AckWait` у
конфігурації, а не намагайтеся підтверджувати «про всяк випадок» одразу після отримання.

---

## Get Consumer Info / Delete Consumer

```
Get Consumer Info
   Stream Name   : "ORDERS"
   Consumer Name : "ORDERS_LIVE"
```

Корисно, щоб перевірити, чи існує споживач, перш ніж підписуватися чи витягувати з нього
повідомлення — а не покладатися на те, що він точно вже створений.

```
Delete Consumer
   Stream Name   : "ORDERS"
   Consumer Name : "OLD_PROCESSOR"
```

Видаляє лише споживача та його позицію читання — повідомлення в самому стрімі не
чіпаються. Часто використовується як «скинути прогрес»: видалити й створити заново з іншим
`DeliverPolicy`.

---

## Приклад: черга завдань (Work Queue)

Повний робочий цикл, що поєднує стрім, retention-політику `WorkQueue` і pull-споживача —
класична черга завдань, де кожне завдання обробляється рівно одним воркером:

```cpp
// Одноразове налаштування
FJetStreamStreamConfig StreamConfig = FJetStreamStreamConfigBuilder()
    .WithName(TEXT("TASKS"))
    .WithSubject(TEXT("tasks.pending"))
    .WithRetention(EJetStreamRetentionPolicy::WorkQueue) // видаляється одразу після Ack
    .Build();
Streams->CreateStream(StreamConfig, [](auto) {});

FJetStreamConsumerConfig ConsumerConfig = FJetStreamConsumerConfigBuilder()
    .WithName(TEXT("WORKER"))
    .WithAckPolicy(EJetStreamAckPolicy::Explicit)
    .WithAckWaitSeconds(120.0f) // достатньо для довгої обробки
    .Build();
Consumers->CreateConsumer(TEXT("TASKS"), ConsumerConfig, [](auto) {});

// Постановка завдання (з будь-якого місця гри)
Publisher->Publish(TEXT("tasks.pending"), TaskJson, [](auto) {});

// Цикл воркера (наприклад, кожні кілька секунд, чи одразу після завершення попереднього)
void AWorker::PullNextBatch()
{
    Consumers->PullMessages(TEXT("TASKS"), TEXT("WORKER"), 5, 10.0f,
        [this](TJetStreamResult<TArray<FJetStreamMessage>> Result)
        {
            for (const FJetStreamMessage& Task : Result.Value)
            {
                const bool bOk = ProcessTask(Task.NatsMsg.Data);
                if (bOk)
                {
                    Consumers->AckMessage(Task);
                }
                else
                {
                    Consumers->NakMessage(Task); // спробувати ще раз пізніше
                }
            }
            PullNextBatch(); // наступна порція
        });
}
```

Якщо кілька воркерів одночасно тягнуть з одного pull-споживача — кожне повідомлення дістанеться
лише одному з них: JetStream гарантує, що те саме повідомлення не буде видано двом запитам
`Pull Messages` одночасно.

---

**Далі:** [10. JetStream: Key-Value](10-JetStream-KeyValue.md)
