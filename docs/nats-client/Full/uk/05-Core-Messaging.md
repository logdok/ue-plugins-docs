*[🇬🇧 English](../en/05-Core-Messaging.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 5. Основний обмін повідомленнями

Цей розділ — про режим Core: підключення, публікацію, підписку та запит-відповідь. Якщо
терміни «subject», «wildcard» чи «request/reply» ще незнайомі — спершу прочитайте
[1. Вступ до NATS](01-Introduction.md).

---

## Три способи скористатися плагіном

Усі три дають однаковий набір операцій (`Publish`, `Subscribe`, `Request Async` тощо) —
різниця лише в тому, хто керує життям з'єднання.

| | **NATS Client Subsystem** | **NATS Client Component** | **NATS Core Client** (`UNatsClient`) |
|---|---|---|---|
| Що це | `GameInstanceSubsystem` — один екземпляр на всю гру | `ActorComponent` — один на актора | Голий об'єкт з'єднання |
| Живе | Усю гру, переживає зміну рівнів | Поки живий актор-власник | Поки ви самі його тримаєте |
| Бере налаштування з | Project Settings | Власних властивостей компонента | Параметрів, які передасте самі |
| Коли обирати | **Типовий вибір.** Один спільний потік повідомлень на всю гру | З'єднання логічно належить конкретному актору (наприклад, окремий NPC-сервіс) | Просунуті сценарії: власне управління життєвим циклом, кілька незалежних з'єднань одночасно |

**Рекомендація за замовчуванням — підсистема.** Решта прикладів у цьому розділі показані на
ній; для компонента й голого клієнта сигнатури функцій ідентичні.

> **Автоперепідключення є лише в підсистемі й компоненті.** Голий `NATS Core Client`
> сам собою після розриву зв'язку нічого не робить — просто повідомляє про це подією
> `On Disconnected` і чекає на ваш наступний виклик `Connect`. Якщо обираєте голий клієнт
> заради повного контролю — логіку повторних спроб доведеться реалізувати самостійно.

```
Get Game Instance Subsystem (Nats Client Subsystem)
```

— одна нода, доступна з будь-якого графа. Створювати об'єкт чи зберігати його у змінній не
треба: підсистема вже існує на момент старту гри й повертає той самий екземпляр щоразу.

---

## Підключення

### Connect To Server

Підключається, використовуючи значення з [Project Settings](03-Configuration.md) —
адресу, порт і облікові дані. Найпростіший спосіб, коли конфігурація одна на весь проєкт.

```
Event BeginPlay
   │
   └─► Get Game Instance Subsystem (Nats Client Subsystem)
          │
          └─► Connect To Server
```

### Connect

Підключення з явно вказаною адресою й портом, в обхід Project Settings:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Connect
          Server URL     : "192.168.1.100"
          Port           : 4222
          b Non Blocking : false
```

`b Non Blocking` (типово `false`) впливає лише на те, як створюється сокет; на швидкість
самого підключення це не впливає — `On Connected` у будь-якому разі спрацює асинхронно,
коли завершиться TCP- та NATS-рукостискання.

### Connect With Credentials

Те саме, але з явними обліковими даними — типовий вибір, коли дані приходять під час
виконання (докладно й з прикладом — [3. Налаштування](03-Configuration.md#де-задати-облікові-дані)):

```cpp
UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();

FNatsCredentials Credentials;
Credentials.AuthType = ENatsAuthType::Basic;
Credentials.Username = TEXT("player123");
Credentials.Password = TEXT("secret");

Nats->ConnectWithCredentials(TEXT("nats.example.com"), 4222, Credentials);
```

### Повторний Connect під час підключення

Виклик `Connect`/`Connect To Server`, поки попереднє підключення ще триває або вже активне,
повертає `false` і нічого не робить — другого паралельного з'єднання не виникає. Щоб
підключитися до іншого сервера, спершу викличте `Disconnect`.

### Disconnect

```
Disconnect
   Reason : "Гравець вийшов у головне меню"
```

`Reason` — довільний текст для логів і для обробника `On Disconnected`, якщо ви хочете
розрізняти навмисне відключення від розриву зв'язку. Зупиняє фоновий потік, закриває сокет,
очищає всі підписки. Якщо було увімкнено `b Auto Reconnect` — цикл повторних спроб теж
зупиняється.

### IsConnected

Доступний на всіх трьох точках входу — підсистемі, компоненті й голому клієнті:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Is Connected ──► (bool) ──► Branch
```

```cpp
if (Nats->IsConnected())
{
    Nats->Publish(TEXT("game.events.ping"), TEXT(""));
}
```

### Get Connection State (лише NATS Core Client)

Той самий стан, але з подробицями — доступний лише на голому `NATS Core Client`
(`UNatsClient`), не на підсистемі чи компоненті. Нода є і в Blueprint (`Is Connected` →
чистий пін `Get Connection State` на самому об'єкті клієнта), і в C++:

```cpp
UNatsClient* Client = NewObject<UNatsClient>();
// ...
if (Client->GetConnectionState() == ENatsConnectionState::Connecting)
{
    // TCP-з'єднання або NATS-рукостискання ще триває
}
```

| Значення | Що означає |
|---|---|
| `Disconnected` | Немає активного з'єднання. Причини: ще не викликали `Connect`, підключення розірвано, чи не вдалося з першої спроби |
| `Connecting` | TCP-з'єднання встановлюється або триває NATS-рукостискання (зазвичай менше секунди) |
| `Connected` | Повністю готовий: можна публікувати, підписуватись, робити запити |

---

## Публікація

### Publish

```
Publish
   Subject  : "game.events.player.join"
   Data     : "{\"player\":\"Alice\"}"
   Reply To : (порожньо)
```

`Data` — звичайний текст, переданий як UTF-8: JSON, plain text, XML — будь-яке текстове
представлення. `Reply To` заповнюється лише для власноручної реалізації запит-відповіді
(зазвичай замість цього використовують `Request Async` — [нижче](#requestreply)). Повертає
`true`, якщо повідомлення успішно поставлено в чергу на відправку; `false` — якщо клієнт не
підключений.

> Для бінарних даних (зображення, стиснені дані, серіалізовані структури) `Publish` не
> підходить — текстовий шлях може спотворити байти, які не є коректним UTF-8. Дивіться
> [6. Бінарні дані](06-Binary-Data.md).

### Publish With Headers

Заголовки — пари ключ-значення поруч із даними: тип вмісту, ідентифікатор трасування,
пріоритет — усе, що логічно відокремити від самого тіла повідомлення.

```
Make Map (String → String)
   ["Content-Type"] = "application/json"
   ["Trace-Id"]      = "abc-123-xyz"
   │
   └─► Publish With Headers
          Subject  : "game.events.player.join"
          Data     : "{\"player\":\"Alice\"}"
          Headers  : (мапа зверху)
```

```cpp
TMap<FString, FString> Headers;
Headers.Add(TEXT("Content-Type"), TEXT("application/json"));
Headers.Add(TEXT("Trace-Id"), TEXT("abc-123-xyz"));

Nats->Publish(TEXT("game.events.player.join"), Payload); // без заголовків
// або, з тим самим NatsClient, доступним через Subsystem/Component:
NatsClientInstance->PublishWithHeaders(Subject, Payload, Headers);
```

> **Заголовки вимагають сервера NATS 2.2.0 і новіше.** Якщо сервер їх не підтримує,
> `Publish With Headers` поверне `false` — перевіряйте це, якщо працюєте з дуже старими
> розгортаннями сервера.

Отримувач читає заголовки з поля `Headers` структури `FNatsMessage` — [нижче](#структура-повідомлення-fnatsmessage).

---

## Підписка

### Subscribe / Unsubscribe

```
Subscribe
   Subject : "chat.lobby.*"
```

Підтримує обидва символи шаблону — `*` (один токен) і `>` (один або більше токенів до
кінця), докладно розібрані в [1. Вступ до NATS](01-Introduction.md#subject-адреса-повідомлення).
Кожен виклик `Subscribe` реєструє окрему підписку; щоб перестати отримувати повідомлення на
цей самий subject, викличте:

```
Unsubscribe
   Subject : "chat.lobby.*"
```

`Subject` для `Unsubscribe` має **точно** збігатися з рядком, переданим у `Subscribe` —
включно з зірочками й символом `>`, якщо вони там були.

### Структура повідомлення (`FNatsMessage`)

Усі отримані повідомлення — і у `On Message Received`, і в результаті `Request Async` —
приходять цією структурою:

| Поле | Тип | Опис |
|---|---|---|
| `Subject` | `FString` | Точний subject, на який опубліковано повідомлення (не шаблон підписки) |
| `Data` | `FString` | Вміст повідомлення, декодований як UTF-8-текст |
| `Reply To` | `FString` | Subject для відповіді, якщо публікатор його вказав (порожньо в звичайних повідомленнях) |
| `Headers` | `TMap<FString, FString>` | Заголовки, якщо публікація йшла через `Publish With Headers` |
| `Payload` | `TArray<uint8>` | Ті самі дані, але як точні байти — див. [6. Бінарні дані](06-Binary-Data.md) |

### Подія On Message Received

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Bind Event to On Message Received
            │
            └─► (custom event)
                    Message
                       │
                       ├─► Break Nats Message
                       │       ├─ Subject
                       │       ├─ Data
                       │       └─ Headers
                       │
                       └─► Switch on String (Subject)
```

Подія спрацьовує **для всіх** повідомлень Core, отриманих на будь-яку з активних підписок —
якщо ви підписані на кілька subject'ів, розрізняйте їх усередині обробника за `Message.Subject`
(`Switch on String`, `Contains`, чи власна логіка маршрутизації).

> Подія завжди спрацьовує в ігровому потоці, незалежно від того, звідки прийшло
> повідомлення мережею — усередині безпечно звертатися до акторів і віджетів.

---

## Request/Reply

Патерн описано концептуально в [1. Вступ до NATS](01-Introduction.md#requestreply-коли-потрібна-відповідь);
тут — як ним скористатися.

### Request Async

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Request Async
          Subject         : "service.users.get"
          Data            : "{\"id\":123}"
          Timeout Seconds : 5.0
          │
          Callback ──► (bSuccess, Response)
                            │
                            ├─ true  → Print String (Response)
                            └─ false → Print String (Response)   ← містить причину: "Timeout", "Not connected" тощо
```

Плагін сам створює тимчасову скриньку (`_INBOX.<guid>`), підписується на неї, публікує
запит із `reply-to`, встановленим на цю скриньку, і чекає відповіді — усе це за одним
викликом. Якщо відповідь не прийшла за `Timeout Seconds`, `Callback` спрацює з `bSuccess =
false`, а `Response` міститиме `"Timeout"`.

C++:

```cpp
Nats->RequestAsync(TEXT("service.users.get"), TEXT("{\"id\":123}"), 5.0f,
    [](bool bSuccess, const FString& Response)
    {
        if (bSuccess)
        {
            UE_LOG(LogTemp, Log, TEXT("Отримано: %s"), *Response);
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Запит не вдався: %s"), *Response);
        }
    });
```

Колбек завжди викликається в ігровому потоці — і в `Request Async` (BP), і в
`RequestAsync` (C++-делегат).

### Сторона відповідача

Жодної спеціальної ноди не потрібно — відповідач просто підписується на subject запиту й
публікує відповідь у `Message.ReplyTo`:

```
On Message Received (Message → Subject == "service.users.get")
   │
   └─► Publish
          Subject : Message → Reply To
          Data    : "{\"id\":123,\"name\":\"Alice\",\"level\":5}"
```

Якщо відповідач нічого не публікує назад — запитувач просто отримає тайм-аут, це нормальна,
очікувана поведінка NATS, а не помилка плагіна.

### RequestAsyncWithHeaders (лише C++)

Той самий запит, але з заголовками — наприклад, для передавання токена авторизації запиту
окремо від тіла:

```cpp
TMap<FString, FString> Headers;
Headers.Add(TEXT("Authorization"), TEXT("Bearer abc123"));

Nats->RequestAsyncWithHeaders(TEXT("service.users.get"), TEXT("{\"id\":123}"), Headers, 5.0f,
    [](bool bSuccess, const FString& Response)
    {
        // ...
    });
```

Для Blueprint-версії з заголовками — використовуйте `Publish With Headers` на боці
відповідача разом зі звичайним `Request Async` на боці запитувача; NATS не розрізняє «запит
із заголовками» й «звичайний запит із заголовками» — це одна й та сама публікація з
`reply-to`.

### Отримати повний об'єкт відповіді (лише C++)

`RequestAsync` повертає лише `Data` відповіді рядком. Якщо потрібні ще й заголовки чи
`ReplyTo` самої відповіді (актуально, наприклад, для деяких сценаріїв JetStream pull-
споживачів — [9. JetStream: споживачі](09-JetStream-Consumers.md)), скористайтеся
`RequestAsyncWithMessage`:

```cpp
Nats->RequestAsyncWithMessage(TEXT("service.users.get"), TEXT("{\"id\":123}"), 5.0f,
    [](bool bSuccess, const FNatsMessage& ResponseMessage)
    {
        if (bSuccess)
        {
            UE_LOG(LogTemp, Log, TEXT("Дані: %s, Заголовки: %d"),
                *ResponseMessage.Data, ResponseMessage.Headers.Num());
        }
    });
```

---

## Події з'єднання

| Подія | Коли спрацьовує |
|---|---|
| **On Connected** | Одразу після успішного NATS-рукостискання. Тут — правильне місце для `Subscribe` |
| **On Disconnected** | З'єднання розірвано: чи вашим викликом `Disconnect`, чи через мережевий збій, чи через зупинку сервера. `Reason` описує причину |
| **On Error** | Помилка на будь-якому етапі — від невдалого підключення до серверної помилки протоколу |

Усі три завжди спрацьовують в ігровому потоці.

Якщо `b Auto Reconnect` увімкнено (типово так — [3. Налаштування](03-Configuration.md#перепідключення)),
після `On Disconnected` (не спричиненого вашим `Disconnect`) підсистема сама почне повторні
спроби підключення до того самого сервера; повторний `On Connected` спрацює, щойно
з'єднання відновиться — підписки, зроблені в обробнику `On Connected`, спрацюють знову
автоматично, оскільки сервер не пам'ятає підписки з розірваного з'єднання.

---

**Далі:** [6. Бінарні дані](06-Binary-Data.md)
