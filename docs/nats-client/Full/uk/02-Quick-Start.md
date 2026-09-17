*[🇬🇧 English](../en/02-Quick-Start.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 2. Швидкий старт

Мета розділу — за десять хвилин підключитися до сервера, опублікувати перше повідомлення й
отримати його назад, не написавши жодного рядка C++.

---

## Крок 0. Сервер для перевірки

Плагін — лише клієнт: йому потрібен окремий NATS-сервер, до якого підключатися. Якщо у вас
його ще немає, найшвидший спосіб підняти локальний — одна команда з Docker:

```bash
docker run -p 4222:4222 -p 8222:8222 nats:latest -js
```

Прапорець `-js` вмикає JetStream — знадобиться в розділах 6–9. Порт `4222` — це сам
протокол NATS, `8222` — сторінка моніторингу (`http://localhost:8222/varz`), корисна для
перевірки, що сервер справді піднявся.

Для постійнішого локального оточення (з JetStream, що зберігається між перезапусками) у
плагіні вже є готовий `docker-compose.yml` — детальніше в розділі
[12. Тестування та локальний сервер](12-Testing-And-Local-Server.md).

---

## Крок 1. Установлення

1. Скопіюйте теку `NatsClient/` у каталог `Plugins/` вашого проєкту (або встановіть плагін
   через Fab/Epic Games Launcher — тоді цей крок не потрібен).
2. Перегенеруйте файли проєкту та зберіть його.
3. Переконайтеся, що плагін увімкнено: **Edit → Plugins → Networking → NATS Message Broker
   Client**.

---

## Крок 2. Налаштування підключення

**Project Settings → Plugins → NATS Messaging Client**

Для локального сервера з кроку 0 налаштування за замовчуванням уже підходять:

| Поле | Значення за замовчуванням | Коли міняти |
|---|---|---|
| **Server URL** | `127.0.0.1` | Віддалений сервер — впишіть адресу чи ім'я хоста |
| **Port** | `4222` | Сервер слухає інший порт |
| **Credentials → Auth Type** | `None` | Сервер вимагає авторизації — див. [3. Налаштування](03-Configuration.md#облікові-дані) |

Повний опис усіх полів сторінки — у розділі [3. Налаштування](03-Configuration.md).

---

## Крок 3. Перша підписка й публікація з Blueprint

```
Event BeginPlay
   │
   ├─► Get Game Instance Subsystem (Nats Client Subsystem) ──┐
   │                                                          │
   │                                                          ├─► Connect To Server
   │                                                          │
   │                                                          ├─► Bind Event to On Message Received
   │                                                          │        │
   │                                                          │        └─► Print String (Message → Data)
   │                                                          │
   │                                                          └─► Bind Event to On Connected
   │                                                                   │
   │                                                                   └─► Subscribe
   │                                                                          Subject : "game.events.>"
```

**Get Game Instance Subsystem** — стандартна нода Unreal: правий клік у графі → введіть
«Get Game Instance Subsystem» → оберіть клас **Nats Client Subsystem**. Підсистема — це
єдиний рекомендований вхід до плагіна: вона живе всю гру, тримає з'єднання й сама
перепідключається при розриві. Створювати клієнта вручну чи зберігати його у змінній не
треба.

Підписуйтесь **саме на `On Connected`**, а не одразу після `Connect To Server`: сам виклик
лише починає підключення (TCP-рукостискання й handshake NATS займають частку секунди), і
`Subscribe`, викликаний до фактичного з'єднання, нічого не зробить.

Тепер опублікуємо повідомлення — на цей самий subject, щоб отримати власне повідомлення
назад:

```
[будь-яка подія, наприклад натискання кнопки]
   │
   └─► Get Game Instance Subsystem (Nats Client Subsystem)
          │
          └─► Publish
                 Subject : "game.events.player.join"
                 Data    : "{\"player\":\"Alice\"}"
```

Натисніть кнопку — у логах з'явиться Print String із щойно опублікованими даними: підписка
на `game.events.>` захопила subject `game.events.player.join` за правилом шаблону `>`
(див. [1. Вступ до NATS](01-Introduction.md#subject-адреса-повідомлення)).

---

## Крок 4. Той самий приклад у C++

```cpp
#include "NatsClientSubsystem.h"

void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();

    UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();

    Nats->OnConnected.AddDynamic(this, &AMyGameMode::HandleConnected);
    Nats->OnMessageReceived.AddDynamic(this, &AMyGameMode::HandleMessage);

    Nats->ConnectToServer();
}

void AMyGameMode::HandleConnected()
{
    UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
    Nats->Subscribe(TEXT("game.events.>"));
}

void AMyGameMode::HandleMessage(const FNatsMessage& Message)
{
    UE_LOG(LogTemp, Log, TEXT("Отримано %s: %s"), *Message.Subject, *Message.Data);
}

void AMyGameMode::OnPlayerJoined(const FString& PlayerName)
{
    UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
    Nats->Publish(TEXT("game.events.player.join"),
        FString::Printf(TEXT("{\"player\":\"%s\"}"), *PlayerName));
}
```

`HandleConnected` і `HandleMessage` мають бути `UFUNCTION()`, щоб `AddDynamic` міг на них
посилатися — це стандартна вимога Unreal до динамічних делегатів, не особливість плагіна.

---

## Крок 5. Перевірка прямо з редактора

На тій же сторінці **Project Settings → Plugins → NATS Messaging Client** вгорі є дві
кнопки:

| Кнопка | Що робить |
|---|---|
| **Test Connection** | Підключається до сервера з поточних налаштувань і показує сповіщення: вдалося чи ні |
| **Test JetStream** | Те саме, плюс перевіряє, що на сервері увімкнено JetStream |

Зручно перевірити конфігурацію одразу після кроку 2, ще до написання будь-якого графа.

---

## Крок 6. Request/Reply одним викликом

Коли потрібна саме відповідь, а не просто підписка — патерн, описаний у
[1. Вступ до NATS](01-Introduction.md#requestreply-коли-потрібна-відповідь) — плагін ховає всю механіку скриньок
за однією нодою:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Request Async
          Subject          : "service.users.get"
          Data             : "{\"id\":123}"
          Timeout Seconds  : 5.0
          │
          Callback → (bSuccess, Response)
                         │
                         ├─ true  → Print String (Response)
                         └─ false → Print String ("Тайм-аут або помилка")
```

Відповідач — окремий клієнт (можливо, ваш dedicated server), який підписаний на
`service.users.get` і публікує відповідь у `Message.ReplyTo`:

```
On Message Received (Subject == "service.users.get")
   │
   └─► Publish
          Subject : Message → Reply To
          Data    : "{\"id\":123,\"name\":\"Alice\"}"
```

Повний розбір, включно з варіантом для бінарних даних, — у
[5. Основний обмін повідомленнями](05-Core-Messaging.md#requestreply).

---

## Крок 7. Перший стрім JetStream

Публікація й підписка вище — режим Core: немає підписника в момент публікації, немає й
повідомлення. Якщо дані не можна втрачати, потрібен JetStream —
[1. Вступ до NATS](01-Introduction.md#core-проти-jetstream-два-режими-одного-сервера).

```
[після On Connected]
   │
   └─► Get JetStream
          │
          └─► Get Streams Manager
                 │
                 └─► Create Stream
                        Config → Name     : "ORDERS"
                        Config → Subjects : ["orders.>"]
                        │
                        └─ bSuccess → Get Publisher → Publish Message
                                         Subject : "orders.created"
                                         Data    : "{\"id\":1}"
```

Тепер `orders.created` зберігається на сервері незалежно від того, чи є в цю мить хтось
підписаний — прочитати його можна навіть через годину, створивши споживача. Повний розбір
стрімів, споживачів (push і pull) і Key-Value сховища — у розділах 6–9.

---

## Типові перешкоди на старті

| Симптом | Причина |
|---|---|
| `On Connected` ніколи не спрацьовує | Сервер недоступний за вказаною адресою/портом, або запущений без `-js`, якщо очікуєте JetStream. Перевірте кнопкою **Test Connection** |
| `Subscribe` нічого не приносить | Підписка викликана до `On Connected` — `IsConnected()` на момент виклику був `false` |
| Publish повертає `false` | Клієнт ще не підключений. Перевірте `IsConnected` перед публікацією |
| `Request Async` завжди повертає тайм-аут | Відповідач не публікує назад у `Message.ReplyTo`, або підписаний не на той subject |
| Дані з кирилицею чи емодзі виглядають биті на боці приймача | Малоймовірно з цією версією плагіна (2.1) — байти передаються точно; перевірте, що приймач сам не обрізає/перекодовує рядок |

---

**Далі:** [3. Налаштування](03-Configuration.md)
