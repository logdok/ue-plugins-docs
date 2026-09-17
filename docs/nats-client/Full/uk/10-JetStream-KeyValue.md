*[🇬🇧 English](../en/10-JetStream-KeyValue.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 10. JetStream: Key-Value

Key-Value (KV) — сховище «ключ → значення» поверх JetStream; концептуально розібрано в
[1. Вступ до NATS](01-Introduction.md#key-value-сховище-поверх-стріму). Ключ у KV — це,
по суті, subject `$KV.<бакет>.<ключ>` у прихованому стрімі, тож вимоги до символів у ключі —
ті самі, що й у subject: крапка розділяє «рівні» ключа (`player.42.stats` — валідний ключ
із крапками), і сам ключ не повинен містити пробілів.

---

## Бакети

### CreateBucket / DeleteBucket

```
Get JetStream
   │
   └─► Get Key-Value Store
          │
          └─► Create KV Bucket
                 Config → Bucket  : "PLAYER_STATS"
                 Config → History : 1
                 Config → Storage : File
```

```cpp
UNatsKVStoreImpl* KV = Nats->GetJetStream()->KeyValue();

FJetStreamKVConfig Config = FJetStreamKVConfigBuilder()
    .WithBucket(TEXT("PLAYER_STATS"))
    .WithStorage(EJetStreamStorageType::File)
    .WithHistory(5)
    .Build();

KV->CreateBucket(Config, [](TJetStreamResult<bool> Result) {});
```

Як і `Create Stream`, повторний виклик з існуючим бакетом повертає успіх, а не помилку —
безпечно викликати при кожному старті гри.

### Конфігурація бакета

`FJetStreamKVConfig`:

| Поле | Тип | За замовчуванням | Опис |
|---|---|---|---|
| `Bucket` | `FString` | — | Ім'я бакета, наприклад `"PLAYER_STATS"` |
| `MaxAge` | `int32` | `0` (без обмеження) | Час життя запису, секунди |
| `MaxValueSize` | `int32` | `-1` (ліміт сервера) | Максимальний розмір значення, байти |
| `History` | `int32` | `1` | Скільки версій зберігати на ключ. `1` — лише поточне значення |
| `Storage` | `EJetStreamStorageType` | `Memory` | `Memory` чи `File` — той самий принцип, що й у стрімах |
| `Replicas` | `int32` | `1` | Реплікація в кластері |

```
Delete KV Bucket
   Bucket : "OLD_BUCKET"
```

**Незворотно:** видаляє бакет і всі ключі в ньому.

---

## Базові операції

### KV Put / KV Get

```
KV Put
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
   Value  : "{\"level\":5,\"xp\":1200}"
   │
   └─ bSuccess → Entry → Revision : 1
```

```
KV Get
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
   │
   └─ bSuccess → Entry → Value : "{\"level\":5,\"xp\":1200}"
   └─ !bSuccess → ключа немає (Error Code = Key Not Found)
```

```cpp
KV->Put(TEXT("PLAYER_STATS"), TEXT("player_42"), TEXT("{\"level\":5,\"xp\":1200}"),
    [](TJetStreamResult<FJetStreamKVEntry> Result) {});

KV->Get(TEXT("PLAYER_STATS"), TEXT("player_42"),
    [](TJetStreamResult<FJetStreamKVEntry> Result)
    {
        if (Result.IsSuccess())
        {
            UE_LOG(LogTemp, Log, TEXT("Рівень: %s, ревізія: %d"), *Result.Value.Value, Result.Value.Revision);
        }
    });
```

### KV Delete

```
KV Delete
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
```

> **Відома особливість поточної версії.** Якщо в бакета `History` більше 1, `KV Get` після
> `KV Delete` може повернути **попереднє** значення ключа замість помилки «ключ не
> знайдено» — детальніше в
> [11. Помилки та діагностика](11-Errors-And-Diagnostics.md#kv-delete-у-бакеті-з-історією).
> Для бакетів з `History = 1` (типове значення) поведінка коректна.

### KV List Keys

```
KV List Keys
   Bucket : "PLAYER_STATS"
   │
   └─ bSuccess → Keys : ["player_1", "player_42", "player_99"]
```

---

## Оптимістична конкурентність

### KV Put With Revision

Записати значення, лише якщо ніхто інший не змінив ключ між вашим читанням і записом —
класичне «read-modify-write» без гонок:

```
KV Get (Bucket: "PLAYER_STATS", Key: "player_42")
   │
   └─► Entry → Revision  ───────────┐
                                     │
[змінили дані локально]             │
                                     ▼
KV Put With Revision
   Bucket             : "PLAYER_STATS"
   Key                : "player_42"
   Value              : (оновлені дані)
   Expected Revision  : (Revision з Get вище)
   │
   └─ !bSuccess → хтось інший устиг записати першим — прочитайте заново й повторіть
```

```cpp
KV->Get(TEXT("PLAYER_STATS"), TEXT("player_42"),
    [this](TJetStreamResult<FJetStreamKVEntry> GetResult)
    {
        if (!GetResult.IsSuccess()) return;

        const FString NewValue = ApplyChange(GetResult.Value.Value);

        Nats->GetJetStream()->KeyValue()->PutWithRevision(
            TEXT("PLAYER_STATS"), TEXT("player_42"), NewValue, GetResult.Value.Revision,
            [](TJetStreamResult<FJetStreamKVEntry> PutResult)
            {
                if (!PutResult.IsSuccess())
                {
                    // Конфлікт версій — хтось записав між Get і Put. Повторіть цикл.
                }
            });
    });
```

### KV Create

«Записати, лише якщо ключа ще не існує» — атомарна операція, корисна для розподіленого
блокування чи «перший гравець отримує приз»:

```
KV Create
   Bucket : "LOCKS"
   Key    : "boss_fight_1"
   Value  : "owned_by_session_abc"
   │
   ├─ bSuccess = true  → ключ щойно створено саме вами — ви власник
   └─ bSuccess = false → ключ уже існував — власник хтось інший
```

```cpp
KV->Create(TEXT("LOCKS"), TEXT("boss_fight_1"), SessionId,
    [](TJetStreamResult<FJetStreamKVEntry> Result)
    {
        if (Result.IsSuccess())
        {
            StartBossFight(); // ми перші
        }
        else
        {
            // Хтось інший уже почав бій із цим босом
        }
    });
```

На відміну від «спочатку `KV Get`, потім `KV Put`, якщо не знайдено», `KV Create` атомарна:
навіть якщо два гравці викликають її одночасно, успіх отримає рівно один.

---

## Watch: реагувати на зміни в реальному часі

### KV Watch / KV Stop Watch

```
KV Watch
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
   │
   Callback ──► (Entry) → оновити UI щоразу, коли значення змінюється
```

```cpp
FString WatchId = KV->Watch(TEXT("PLAYER_STATS"), TEXT("player_42"),
    [](const FJetStreamKVEntry& Entry)
    {
        UpdateStatsWidget(Entry.Value);
    });

// коли більше не потрібно (наприклад, закриття віджета)
KV->StopWatch(WatchId);
```

**Ключ можна лишити порожнім**, щоб отримувати зміни **усіх** ключів бакета —
зручно для журналювання чи синхронізації повного стану:

```cpp
KV->Watch(TEXT("PLAYER_STATS"), FString(), // порожній Key = усі ключі
    [](const FJetStreamKVEntry& Entry)
    {
        UE_LOG(LogTemp, Log, TEXT("Ключ %s змінено: %s"), *Entry.Key, *Entry.Value);
    });
```

`Key` також підтримує ті самі шаблони `*`/`>`, що й subject Core-підписки
([1. Вступ до NATS](01-Introduction.md#subject-адреса-повідомлення)) — наприклад
`"player_*.status"` для «статусу будь-якого гравця».

> Кожен виклик `KV Watch` завжди повертає **поточне** значення ключа при змінах, отримане
> окремим зверненням до сервера, — а не сирі байти зі службового повідомлення. Це означає
> невелику затримку між фізичною зміною й викликом колбека, зазвичай непомітну.

---

## History

```
KV History
   Bucket : "PLAYER_STATS"
   Key    : "player_42"
   │
   └─ bSuccess → History : [Entry(rev=1), Entry(rev=2), Entry(rev=3)]  ← від найстарішої до найновішої
```

Працює лише якщо бакет створено з `History` більшим за 1 — інакше повертається лише поточний
запис.

---

## Бінарні значення

Той самий принцип «Bytes-варіант поруч із текстовим», що й у решті плагіна
([6. Бінарні дані](06-Binary-Data.md)):

| Текст | Байти |
|---|---|
| `KV Put` | `KV Put Bytes` |
| `KV Put With Revision` | `KV Put Bytes With Revision` |
| `KV Create` | `KV Create Bytes` |

```cpp
TArray<uint8> IconBytes = ExportIconToBytes();
KV->PutBytes(TEXT("PLAYER_AVATARS"), TEXT("player_42"), IconBytes, [](auto) {});
```

Прочитане значення (`KV Get`, `KV Watch`, `KV History`) завжди містить **обидва**
представлення одночасно — `Value` (текст) і `ValueBytes` (точні байти), незалежно від того,
яким методом його записано:

```
KV Get
   │
   └─ bSuccess → Entry
                    ├─ Value       (текстове представлення)
                    └─ Value Bytes (точні байти — саме це поле для бінарних даних)
```

---

## Приклад: гаряче перезавантаження конфігурації

Типовий сценарій, де KV замінює окремий файл конфігурації, що вимагав би перезапуску:

```cpp
void AGameConfigManager::BeginPlay()
{
    Super::BeginPlay();

    UNatsKVStoreImpl* KV = Nats->GetJetStream()->KeyValue();

    // Прочитати поточне значення одразу при старті
    KV->Get(TEXT("CONFIG"), TEXT("difficulty_multiplier"),
        [this](TJetStreamResult<FJetStreamKVEntry> Result)
        {
            if (Result.IsSuccess())
            {
                ApplyDifficulty(FCString::Atof(*Result.Value.Value));
            }
        });

    // І далі реагувати на будь-яку зміну без перезапуску гри
    KV->Watch(TEXT("CONFIG"), TEXT("difficulty_multiplier"),
        [this](const FJetStreamKVEntry& Entry)
        {
            ApplyDifficulty(FCString::Atof(*Entry.Value));
        });
}
```

Дизайнер міняє значення командою `nats kv put CONFIG difficulty_multiplier 1.5` (чи власною
панеллю адміністрування, яка сама викликає `KV Put`) — усі запущені сервери підхоплюють
зміну протягом секунди, без релізу й без перезапуску.

---

**Далі:** [11. Помилки та діагностика](11-Errors-And-Diagnostics.md)
