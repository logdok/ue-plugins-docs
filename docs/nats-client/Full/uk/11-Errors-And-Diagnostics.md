*[🇬🇧 English](../en/11-Errors-And-Diagnostics.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 11. Помилки та діагностика

Core і JetStream повідомляють про помилки по-різному — Core простим текстом, JetStream
структурованим кодом. Обидва розібрано нижче, разом із типовими причинами й тим, як
подивитися, що насправді відбувається на дроті.

---

## Core: подія On Error

У Core немає окремого коду помилки — лише подія з текстовим описом:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Bind Event to On Error
            │
            └─► (Error: FString) → Print String / залогувати
```

Спрацьовує для помилок на будь-якому рівні: не вдалося встановити TCP-з'єднання, сервер
відхилив автентифікацію, порушено протокол. Розрізняти причину доводиться за текстом
повідомлення — для типових формулювань дивіться [таблицю нижче](#типові-повідомлення-core).

Крім події, кожен виклик (`Publish`, `Subscribe`, `Request Async` тощо) повертає `bool`/
`bSuccess` — перевіряйте його одразу, а не покладайтеся лише на `On Error`, яка описує
загальні збої з'єднання, а не кожен окремий неуспішний виклик.

---

## JetStream: структуровані результати

Кожна операція JetStream повертає щось одне з двох.

### C++: TJetStreamResult&lt;T&gt;

```cpp
Streams->GetStreamInfo(TEXT("ORDERS"), [](TJetStreamResult<FJetStreamStreamInfo> Result)
{
    if (Result.IsSuccess())
    {
        const FJetStreamStreamInfo& Info = Result.Value;
        // ...
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("[%s] %s"), *Result.Error.GetCodeString(), *Result.Error.Message);
    }
});
```

### Blueprint: потрійний делегат

Той самий результат у Blueprint — три піни колбека замість одного об'єднаного типу:

```
Get Stream Info
   │
   └─ Callback ──► (bSuccess, StreamInfo, Error)
                       │
                       ├─ bSuccess = true  → StreamInfo заповнено, Error порожній (Code = None)
                       └─ bSuccess = false → StreamInfo типове/порожнє, Error заповнено
```

### FJetStreamError

| Поле | Опис |
|---|---|
| `Code` | Машинний код — `EJetStreamErrorCode`, [таблиця нижче](#коди-помилок-jetstream) |
| `Message` | Людський опис, придатний для логів |
| `Details` | Додатковий контекст (сира відповідь сервера, коди NATS) |

Допоміжні чисті ноди бібліотеки плагіна:

| Нода | Що робить |
|---|---|
| **Is Success (JetStreamError)** | `Code == None` |
| **Is Error (JetStreamError)** | Протилежне до `Is Success` |
| **Get Error Code String** | `Code` текстом, для логів |
| **To String (JetStreamError)** | Усе поле одним рядком — зручно для `Print String` |

### Коди помилок JetStream

| `EJetStreamErrorCode` | Що сталося |
|---|---|
| `None` | Успіх — окремий код, а не відсутність помилки |
| `NotConnected` | Клієнт NATS не підключений |
| `JetStreamUnavailable` | JetStream не увімкнено на сервері (або ще не підтверджено доступність — див. [7. JetStream: стріми](07-JetStream-Streams.md#чи-доступний-jetstream)) |
| `InvalidParameter` | Некоректний аргумент — наприклад, порожнє ім'я стріму |
| `StreamNotFound` | Немає стріму з такою назвою |
| `ConsumerNotFound` | Немає споживача з такою назвою |
| `StreamAlreadyExists` | Ім'я стріму зайняте іншою конфігурацією |
| `ConsumerAlreadyExists` | Ім'я споживача зайняте іншою конфігурацією |
| `Timeout` | Сервер не відповів за відведений час |
| `PermissionDenied` | Облікові дані дійсні, але дія не дозволена політикою сервера |
| `ServerError` | Сервер повернув помилку, що не підпадає під жоден із кодів вище |
| `SerializationError` | Не вдалося розібрати відповідь сервера як JSON |
| `BucketNotFound` | Немає KV-бакета з такою назвою |
| `KeyNotFound` | Немає такого ключа в бакеті |
| `Unknown` | Непередбачена ситуація |

---

## Типові повідомлення Core

| Симптом | Причина |
|---|---|
| `Failed to connect to NATS server. Reason: ...` в `On Error` | Сервер недоступний за вказаною адресою/портом — файрвол, невірний порт, сервер не запущено |
| `NATS Client is already connecting` / `already connected` | Повторний `Connect`, поки попереднє з'єднання ще активне — спершу `Disconnect` |
| `Cannot publish with headers: Server does not support headers` | Сервер старіший за NATS 2.2.0 — оновіть сервер або не використовуйте заголовки |
| `Publish`/`Subscribe` повертають `false` без запису в `On Error` | Клієнт не підключений на момент виклику — перевірте `Is Connected` |
| `Request Async` завжди завершується з `"Timeout"` | Відповідач не підписаний на потрібний subject, або не публікує у `Message.ReplyTo` — див. [5. Основний обмін повідомленнями](05-Core-Messaging.md#requestreply) |

---

## Типові ситуації JetStream

### `JetStreamUnavailable` одразу після підключення

Нормально: перевірка доступності JetStream асинхронна й займає час після `On Connected`.
Дочекайтеся `On Availability Changed` або перевірте `Is Available` перед першим викликом —
[7. JetStream: стріми](07-JetStream-Streams.md#чи-доступний-jetstream).

### `StreamAlreadyExists` при `Create Stream`

На відміну від помилки, `Create Stream` із **тими самими** параметрами для наявного стріму
повертає успіх — код `StreamAlreadyExists` з'являється, лише коли ви намагаєтеся створити
стрім із тим самим іменем, але **іншою** конфігурацією (наприклад, іншим `Storage`).
Перейменуйте стрім або спершу `Delete Stream`.

### KV Delete у бакеті з історією

Якщо бакет створено з `History` більшим за 1, поточна версія плагіна фізично видаляє лише
останню ревізію ключа при `KV Delete` — і `KV Get` після цього може повернути **попереднє**
значення замість помилки `KeyNotFound`, а `KV History` — показати на один запис менше, ніж
очікувалося. Це відоме обмеження, а не випадкова поведінка коду.

**Що робити зараз:** для бакетів, де важлива саме семантика «видалено назавжди», тримайте
`History = 1` (значення за замовчуванням) — там `KV Delete` поводиться коректно. Якщо
історія потрібна саме для цього бакета — трактуйте `KV Delete` як «прибрати з активного
використання» (наприклад, перевіряйте окреме поле-прапорець у значенні), а не покладайтеся
на те, що ключ фізично зникне.

### Pull Messages повертає порожній масив

Не помилка — означає, що за `Timeout Seconds` не з'явилося жодного нового повідомлення.
Перевіряйте `Result.IsSuccess()`, а не лише непорожність масиву.

---

## Логи

Плагін пише в окремі категорії для кожного шару — вмикайте лише те, що зараз налагоджуєте:

| Категорія | Що логує |
|---|---|
| `LogNatsClient` | Ядро: підключення, публікація, розбір протоколу |
| `LogNatsClientSubsystem` | Підсистема: підключення, перепідключення, маршрутизація JetStream-повідомлень |
| `LogNatsClientComponent` | Те саме для `NATS Client Component` |
| `LogNatsProtocol` | Розбір і збірка кадрів протоколу — найнижчий рівень |
| `LogNatsJetStreamContext` | Ініціалізація JetStream, перевірка доступності |
| `LogJetStreamRequestHandler` | Запити до JetStream API сервера |
| `LogNatsStreamManager` | Операції зі стрімами |
| `LogNatsConsumerManager` | Операції зі споживачами, push/pull доставка |
| `LogNatsJetStreamPublisher` | Публікація в JetStream |
| `LogNatsKVStore` | Операції Key-Value |
| `LogJetStreamSerializer` | Серіалізація/десеріалізація JSON запитів до сервера |

Увімкнути детальний рівень — консольною командою в редакторі чи грі:

```
Log LogNatsClient Verbose
Log LogNatsConsumerManager VeryVerbose
```

Або постійно, у `DefaultEngine.ini`:

```ini
[Core.Log]
LogNatsClient=Verbose
LogNatsConsumerManager=VeryVerbose
```

---

## Кнопки перевірки

**Project Settings → Plugins → NATS Messaging Client**:

| Кнопка | Що перевіряє |
|---|---|
| **Test Connection** | Чи вдається підключитися до сервера з поточних налаштувань |
| **Test JetStream** | Те саме, плюс запит до JetStream API — підтверджує, що JetStream увімкнено |

Результат — миттєве спливаюче сповіщення в редакторі, без потреби запускати гру.

---

## Коли нічого не допомагає

1. Увімкніть `Log LogNatsClient VeryVerbose` (і, якщо стосується JetStream —
   `LogJetStreamRequestHandler VeryVerbose`) та відтворіть один випадок.
2. Перевірте сторінку моніторингу сервера — `http://<адреса>:8222/varz` показує список
   активних з'єднань і базову статистику, `http://<адреса>:8222/jsz` — стан JetStream.
3. Якщо є сумніви саме в даних на дроті — `nats sub "тема.>"` через
   [NATS CLI](https://github.com/nats-io/natscli) в окремому терміналі покаже сирі
   повідомлення незалежно від плагіна: якщо вони видно там, а плагін мовчить — проблема в
   стороні клієнта; якщо не видно й там — проблема в публікації чи в самому сервері.

---

**Далі:** [12. Тестування та локальний сервер](12-Testing-And-Local-Server.md)
