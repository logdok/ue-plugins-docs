*[🇬🇧 English](../en/07-JetStream-Streams.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 7. JetStream: стріми

Стрім — сховище повідомлень; концептуально розібрано в
[1. Вступ до NATS](01-Introduction.md#стрім-stream-де-зберігаються-повідомлення). Цей
розділ — про те, як створювати й керувати стрімами через плагін.

---

## Вхід до JetStream

Усі операції JetStream — стріми, споживачі, публікація, Key-Value — доступні через один
об'єкт-контекст, який плагін отримує автоматично, щойно клієнт підключається:

```
Get Game Instance Subsystem (Nats Client Subsystem)
   │
   └─► Get JetStream
          │
          ├─► Get Streams Manager    ──► [цей розділ]
          ├─► Get Consumers Manager  ──► 9. JetStream: споживачі
          ├─► Get Publisher          ──► 8. JetStream: публікація
          └─► Get Key-Value Store    ──► 10. JetStream: Key-Value
```

```cpp
UNatsJetStreamContext* JS = Nats->GetJetStream();
UNatsStreamManagerImpl* Streams = JS->Streams();
```

### Чи доступний JetStream

Сервер може працювати без увімкненого JetStream (плагін це нормально підтримує — Core
працює незалежно, [1. Вступ до NATS](01-Introduction.md#core-проти-jetstream-два-режими-одного-сервера)).
Перевірити доступність:

```
Get JetStream
   │
   └─► Is Available ──► (bool)
```

Або підписатись на зміну статусу — корисно, якщо сервер піднімається повільніше за гру:

```
Get JetStream
   │
   └─► Bind Event to On Availability Changed
            │
            └─► (bAvailable) → якщо true, тепер можна створювати стріми
```

> Одразу після `On Connected` JetStream ще, як правило, недоступний — перевірка йде
> асинхронно. Якщо потрібні автостворювані стріми без ручного очікування цієї події —
> скористайтеся `Auto Create Streams` у [3. Налаштуваннях](03-Configuration.md#jetstream-автостворення-ресурсів):
> підсистема сама чекає готовності JetStream.

---

## Конфігурація стріму

`FJetStreamStreamConfig` — структура, спільна для створення, оновлення й автостворення з
Project Settings.

| Поле | Тип | За замовчуванням | Опис |
|---|---|---|---|
| `Name` | `FString` | — | Унікальне ім'я, наприклад `"ORDERS"`. Конвенція — ВЕЛИКІ ЛІТЕРИ |
| `Subjects` | `TArray<FString>` | — | Subject'и, які захоплює стрім, з підтримкою шаблонів: `"orders.>"` |
| `Storage` | `EJetStreamStorageType` | `Memory` | `Memory` — швидко, зникає при перезапуску сервера. `File` — переживає перезапуск |
| `Retention` | `EJetStreamRetentionPolicy` | `Limits` | Див. [нижче](#retention-як-довго-жити-повідомленню) |
| `Discard` | `EJetStreamDiscardPolicy` | `Old` | `Old` — витісняти старі повідомлення новими. `New` — відхиляти нові, коли ліміт вичерпано |
| `MaxMsgs` | `int32` | `-1` (без ліміту) | Максимум повідомлень у стрімі |
| `MaxBytes` | `int32` | `-1` (без ліміту) | Максимум сумарного розміру |
| `MaxAge` | `int32` | `0` (без ліміту) | Максимальний вік повідомлення, секунди |
| `MaxMsgSize` | `int32` | `-1` (ліміт сервера) | Максимальний розмір одного повідомлення |
| `Replicas` | `int32` | `1` | Кількість реплік у кластері (потребує NATS-кластера з кількох вузлів) |
| `NoAck` | `bool` | `false` | Вимкнути підтвердження публікації (не рекомендується) |

> `MaxMsgs`/`MaxBytes` рівні `-1` означають «без обмежень» — на продакшн-сервері це
> означає необмежене зростання диска чи пам'яті. Ставте розумний ліміт свідомо, а не
> лишайте типове значення без роздумів.

### Retention: як довго жити повідомленню

| Значення | Поведінка | Типове застосування |
|---|---|---|
| `Limits` (типово) | Повідомлення живуть, доки не вичерпано `MaxMsgs`/`MaxBytes`/`MaxAge` | Журнал подій, метрики, історія |
| `Interest` | Повідомлення видаляються, щойно всі активні споживачі його підтвердили | Дані, потрібні лише поки є хоч один зацікавлений підписник |
| `WorkQueue` | Повідомлення видаляється одразу після підтвердження **будь-яким** споживачем | Черга завдань: кожне завдання обробляється рівно один раз |

### Builder (C++)

У C++ зручніше збирати конфігурацію через `FJetStreamStreamConfigBuilder` — той самий
результат, що й ручне заповнення структури, але з валідацією й без ризику забути обов'язкове
поле:

```cpp
FJetStreamStreamConfig Config = FJetStreamStreamConfigBuilder()
    .WithName(TEXT("ORDERS"))
    .WithSubject(TEXT("orders.>"))
    .WithStorage(EJetStreamStorageType::File)
    .WithRetention(EJetStreamRetentionPolicy::WorkQueue)
    .WithMaxAge(FTimespan::FromDays(30))
    .Build();

if (TOptional<FString> Error = FJetStreamStreamConfigBuilder()
        .WithName(TEXT("ORDERS"))
        .Validate())
{
    UE_LOG(LogTemp, Error, TEXT("Некоректна конфігурація: %s"), **Error);
}
```

`WithSubject` додає один subject до списку; для кількох одразу — `WithSubjects(TArray<FString>)`.
У Blueprint конфігурація завжди заповнюється напряму через `Break`/`Make Jet Stream Stream
Config`, білдер — лише зручність C++.

---

## Операції

### Create Stream

```
Get Streams Manager
   │
   └─► Create Stream
          Config → Name     : "ORDERS"
          Config → Subjects : ["orders.>"]
          Config → Storage  : File
          │
          └─ bSuccess → StreamInfo → State → Messages, Bytes...
```

```cpp
FJetStreamStreamConfig Config;
Config.Name = TEXT("ORDERS");
Config.Subjects.Add(TEXT("orders.>"));
Config.Storage = EJetStreamStorageType::File;

Streams->CreateStream(Config, [](TJetStreamResult<FJetStreamStreamInfo> Result)
{
    if (Result.IsSuccess())
    {
        UE_LOG(LogTemp, Log, TEXT("Стрім готовий: %s"), *Result.Value.Config.Name);
    }
});
```

Якщо стрім із таким ім'ям і конфігурацією вже існує — операція повертає успіх із наявною
інформацією, а не помилку. Це робить `Create Stream` безпечним для виклику щоразу при
старті гри, без попередньої перевірки існування.

### Get Stream Info

```
Get Stream Info
   Stream Name : "ORDERS"
   │
   └─ bSuccess → StreamInfo
                     ├─ Config → налаштування
                     ├─ State  → Messages, Bytes, FirstSeq, LastSeq, NumConsumers
                     └─ Created
```

Найдешевший спосіб перевірити «чи існує такий стрім» і подивитися поточну статистику —
кількість повідомлень, зайнятий розмір, скільки активних споживачів.

### Update Stream

```cpp
FJetStreamStreamConfig Config;
Config.Name = TEXT("ORDERS"); // обов'язково те саме ім'я
Config.MaxMsgs = 100000;      // новий ліміт

Streams->UpdateStream(Config, Callback);
```

Можна змінювати: `Subjects` (додавати нові), `MaxMsgs`/`MaxBytes`/`MaxAge`, `Retention`,
`Discard`. **Не можна** змінити після створення: `Name` і `Storage` — для цього стрім
доведеться видалити й створити заново.

> **Зменшення лімітів застосовується одразу.** Якщо поточна кількість повідомлень
> перевищує новий `MaxMsgs`, зайві видаляються негайно, за правилом `Discard`.

### Delete Stream

```
Delete Stream
   Stream Name : "OLD_EVENTS"
```

**Незворотно:** видаляються всі повідомлення стріму й усі споживачі на ньому. Якщо потрібно
лише почистити старі повідомлення, а не знищити стрім цілком — зменшіть `MaxAge`/`MaxMsgs`
через `Update Stream`.

### List Streams

```
List Streams
   │
   └─ bSuccess → Stream Names : ["ORDERS", "EVENTS", "TELEMETRY"]
```

Повертає лише імена. Деталі кожного — окремим викликом `Get Stream Info`.

---

## Приклад: стрім подій гри

```cpp
void AMyGameMode::SetupEventStream()
{
    UNatsJetStreamContext* JS = Nats->GetJetStream();

    FJetStreamStreamConfig Config = FJetStreamStreamConfigBuilder()
        .WithName(TEXT("GAME_EVENTS"))
        .WithSubject(TEXT("game.events.>"))
        .WithStorage(EJetStreamStorageType::File)
        .WithRetention(EJetStreamRetentionPolicy::Limits)
        .WithMaxAgeSeconds(7 * 24 * 3600) // тиждень
        .Build();

    JS->Streams()->CreateStream(Config, [](TJetStreamResult<FJetStreamStreamInfo> Result)
    {
        if (!Result.IsSuccess())
        {
            UE_LOG(LogTemp, Error, TEXT("Не вдалося створити стрім: %s"), *Result.Error.Message);
        }
    });
}
```

Тепер усе, що публікується на `game.events.*`, зберігається тиждень — незалежно від того,
чи хтось у цю мить підписаний. Наступний крок — прочитати ці повідомлення через споживача:
[9. JetStream: споживачі](09-JetStream-Consumers.md).

---

**Далі:** [8. JetStream: публікація](08-JetStream-Publisher.md)
