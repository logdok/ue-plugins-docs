*[🇬🇧 English](../en/06-Binary-Data.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 6. Бінарні дані

`Publish` і `Data` з попереднього розділу чудово підходять для тексту — JSON, звичайних
рядків. Але не все, що потрібно передати, — текст: зображення, стиснені дані, серіалізовані
структури, аудіо. Для цього в плагіні є паралельний набір операцій, що працює з точними
байтами, — `TArray<uint8>` замість `FString`.

---

## Чому не просто Publish

`FString` в Unreal — це текст, і шлях `Data`/`Publish` десь по дорозі трактує його саме як
текст (UTF-8-кодування на відправці, декодування на прийомі). Для звичайних рядків це
непомітно й правильно. Але довільний бінарний блок — стиснений файл, зображення,
серіалізована структура — це не обов'язково коректний UTF-8: пряме, побайтове
представлення таких даних як тексту може загубити чи спотворити частину вмісту.

**Правило просте:** якщо дані придумав людина мовою (JSON, звичайний рядок, XML) —
підходить `Publish`/`Data`. Якщо дані — вивід серіалізації, стиснення, файл із диска, —
використовуйте `Publish Bytes`/`Payload`.

---

## Відправка: Publish Bytes

```
Publish Bytes
   Subject   : "game.state.snapshot"
   Payload   : (масив байтів)
   Reply To  : (порожньо)
```

```cpp
TArray<uint8> Bytes = SerializeGameState();
Nats->PublishBytes(TEXT("game.state.snapshot"), Bytes);
```

Кожен байт іде на дріт точно таким, яким був у масиві — включно з нульовими байтами
всередині даних. Сигнатура ідентична текстовому `Publish`, лише тип `Data` замінено на
`Payload`.

### Publish Bytes With Headers

Той самий бінарний шлях, але з заголовками — наприклад, щоб вказати формат вмісту поруч із
самими байтами:

```
Make Map (String → String)
   ["Content-Type"] = "application/octet-stream"
   │
   └─► Publish Bytes With Headers
          Subject : "game.state.snapshot"
          Payload : (масив байтів)
          Headers : (мапа зверху)
```

---

## Отримання: поле Payload

Кожне отримане повідомлення (і в `On Message Received`, і в результаті запиту) уже містить
байти — конвертувати нічого не треба, `Payload` заповнюється завжди, незалежно від того,
яким методом дані було опубліковано:

```
On Message Received
   │
   └─► Break Nats Message
           │
           └─ Payload ──► (використати як TArray<uint8>)
```

```cpp
void AMyActor::HandleMessage(const FNatsMessage& Message)
{
    if (Message.Subject == TEXT("game.state.snapshot"))
    {
        ApplyGameState(Message.Payload);
    }
}
```

`Data` теж заповнюється — тим самим вмістом, декодованим як UTF-8-текст. Для справді
бінарних даних це поле не має сенсу (текстове представлення довільних байтів зазвичай
нечитабельне чи навіть частково втрачене) — просто ігноруйте його й використовуйте
`Payload`.

---

## Текст ↔ байти вручну

Іноді потрібно явно перетворити рядок на байти (наприклад, щоб покласти текстовий JSON
поруч із бінарним блоком у одному повідомленні власного формату) чи навпаки. Дві чисті
(`Pure`) ноди в бібліотеці плагіна:

```
"Привіт" ──► String To UTF-8 Bytes ──► (TArray<uint8>)
```

```
(TArray<uint8>) ──► UTF-8 Bytes To String ──► "Привіт"
```

```cpp
TArray<uint8> Bytes = UNatsBlueprintLibrary::StringToUtf8Bytes(TEXT("Привіт"));
FString Text = UNatsBlueprintLibrary::Utf8BytesToString(Bytes);
```

Обидві коректно працюють з нелатинськими символами: кожен символ кирилиці в UTF-8 займає
два байти, і перетворення в обидва боки точне — те, що ви записали, ви й прочитаєте.

---

## Request/Reply з бінарними даними

Той самий патерн запит-відповідь, що й у [5. Основний обмін повідомленнями](05-Core-Messaging.md#requestreply),
але для бінарного вмісту — відповідь повертається повною структурою `FNatsMessage`, а не
лише текстом, бо саме в `Payload` міститься сенс відповіді:

```
Request Bytes Async
   Subject         : "service.avatar.generate"
   Payload         : (вхідні параметри як байти)
   Timeout Seconds : 10.0
   │
   Callback ──► (bSuccess, Response)
                    │
                    ├─ true  → Response → Payload  (згенероване зображення)
                    └─ false → Response → Data     (причина: "Timeout" тощо)
```

```cpp
Nats->RequestBytesAsync(TEXT("service.avatar.generate"), RequestBytes, 10.0f,
    [](bool bSuccess, const FNatsMessage& Response)
    {
        if (bSuccess)
        {
            SaveAvatarTexture(Response.Payload);
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Не вдалося: %s"), *Response.Data);
        }
    });
```

При невдачі (тайм-аут, немає з'єднання) `Response.Data` містить причину текстом, а
`Response.Payload` — порожній; це узгоджено з тим, як `Request Async` повертає причину в
`Response` при `bSuccess = false`.

---

## Практичні приклади

### Надіслати збереження гри

```cpp
TArray<uint8> SaveData;
UGameplayStatics::SaveGameToMemory(SaveGameObject, SaveData);
Nats->PublishBytes(TEXT("player.save.upload"), SaveData);
```

### Надіслати текстуру (наприклад, скріншот)

```
Export To Bytes (Render Target)
   │
   └─► Publish Bytes
          Subject : "screenshots.upload"
          Payload : (з Export To Bytes)
```

### Запакувати кілька значень в один бінарний блок (C++)

Коли потрібно надіслати кілька полів компактніше за JSON — власна двійкова серіалізація
через `FMemoryWriter`/`FMemoryReader`, стандартний спосіб Unreal:

```cpp
// Відправник
TArray<uint8> Bytes;
FMemoryWriter Writer(Bytes);
int32 PlayerId = 42;
FVector Position = GetActorLocation();
Writer << PlayerId;
Writer << Position;

Nats->PublishBytes(TEXT("game.player.position"), Bytes);

// Отримувач
void AMyActor::HandleMessage(const FNatsMessage& Message)
{
    if (Message.Subject == TEXT("game.player.position"))
    {
        FMemoryReader Reader(Message.Payload);
        int32 PlayerId;
        FVector Position;
        Reader << PlayerId;
        Reader << Position;
    }
}
```

Порядок полів на запис і на читання має точно збігатися — це низькорівнева побайтова
серіалізація, без самоопису формату.

---

## JetStream і Key-Value з бінарними даними

Той самий принцип «Bytes-варіант поруч із текстовим» діє і для JetStream:

- **JetStream Publisher** — `Publish Bytes`, `Publish Bytes With Headers`,
  `Publish Bytes With Message ID`, `Publish Bytes With Expected Sequence` — детально в
  [8. JetStream: публікація](08-JetStream-Publisher.md#bytes-варіанти-публікації).
- **Key-Value** — `KV Put Bytes`, `KV Put Bytes With Revision`, `KV Create Bytes`, а
  прочитане значення повертається одразу і текстом (`Value`), і байтами (`ValueBytes`) —
  детально в [10. JetStream: Key-Value](10-JetStream-KeyValue.md#бінарні-значення).

---

**Далі:** [7. JetStream: стріми](07-JetStream-Streams.md)
