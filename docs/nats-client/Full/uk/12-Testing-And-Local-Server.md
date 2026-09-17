*[🇬🇧 English](../en/12-Testing-And-Local-Server.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 12. Тестування та локальний сервер

---

## Локальний сервер, що йде з плагіном

У теці плагіна `Server/nats/` лежить готовий набір файлів для Docker Compose — не потрібно
шукати образ і вигадувати конфігурацію самостійно.

```
Plugins/NatsClient/Server/nats/
├── docker-compose.yml
├── nats.conf          ← активна конфігурація (порт 4222, JetStream увімкнено)
├── sample nats.conf    ← приклад з автентифікацією та кластером, для довідки
├── start-nats.sh        ← запуск на macOS/Linux
└── start-nats.ps1       ← запуск на Windows
```

Запуск:

```bash
cd Plugins/NatsClient/Server/nats
./start-nats.sh        # macOS/Linux
# або
.\start-nats.ps1        # Windows (PowerShell)
```

Обидва скрипти — це просто `docker-compose up -d`; Docker має бути встановлений заздалегідь.
Сервер піднімається на `127.0.0.1:4222` — саме адреса й порт за замовчуванням у
[Project Settings](03-Configuration.md#зєднання).

### Активна конфігурація (`nats.conf`)

| Параметр | Значення |
|---|---|
| Клієнтський порт | `4222` |
| Порт моніторингу | `8222` |
| JetStream | Увімкнено, `store_dir: /data` (примонтовано як том Docker, дані переживають перезапуск контейнера) |
| Ліміт розміру повідомлення | `1MB` (`max_payload`) |
| Автентифікація | Вимкнено (закоментована в файлі) |

### Перевірка, що сервер живий

```bash
curl http://localhost:8222/varz | grep version
curl http://localhost:8222/jsz
```

Друга команда покаже статистику JetStream — якщо у відповіді є поле `"streams"`, JetStream
точно увімкнено. Те саме, без термінала, — кнопка **Test JetStream** у
[Project Settings](02-Quick-Start.md#крок-5-перевірка-прямо-з-редактора).

### Увімкнути автентифікацію локально

`sample nats.conf` — готовий приклад із трьома користувачами й правами доступу по
subject'ах (окремо для «гравця» й «сервера»), а також приклад секції кластера. Щоб
скористатися: скопіюйте потрібні секції в `nats.conf` (або підмініть файл цілком) і
перезапустіть контейнер:

```bash
docker-compose restart
```

Далі в грі — відповідні `Auth Type`/`Username`/`Password` у
[3. Налаштуваннях](03-Configuration.md#облікові-дані).

### Зупинка та очищення

```bash
docker-compose down          # зупинити
docker-compose down -v       # зупинити і стерти дані JetStream
```

---

## Тестування власної інтеграції

### Ручна перевірка з окремого термінала

Найшвидший спосіб переконатися, що дані на дроті саме такі, як очікуєте, — незалежний
клієнт [NATS CLI](https://github.com/nats-io/natscli), що не залежить від Unreal чи від
плагіна взагалі:

```bash
# В одному терміналі — слухати все
nats sub "game.events.>"

# В іншому — опублікувати вручну
nats pub game.events.test "перевірка зв'язку"
```

Якщо повідомлення з'явилося в підписці CLI, але не в грі — проблема на боці клієнта (subject,
підписка, обробник). Якщо не з'явилося ніде — проблема в самій публікації чи в сервері.

### Ізольований тестовий актор

Для перевірки конкретного сценарію зручно мати окремого, мінімального актора, не пов'язаного
з рештою логіки гри, — підписатися, надрукувати все, що прийшло, і легко видалити після
перевірки:

```cpp
// Мінімальний тестовий актор
UCLASS()
class ANatsSmokeTestActor : public AActor
{
    GENERATED_BODY()

protected:
    virtual void BeginPlay() override
    {
        Super::BeginPlay();

        UNatsClientSubsystem* Nats = GetGameInstance()->GetSubsystem<UNatsClientSubsystem>();
        Nats->OnMessageReceived.AddDynamic(this, &ANatsSmokeTestActor::HandleMessage);
        Nats->OnConnected.AddDynamic(this, &ANatsSmokeTestActor::HandleConnected);
        Nats->ConnectToServer();
    }

    UFUNCTION()
    void HandleConnected()
    {
        GetGameInstance()->GetSubsystem<UNatsClientSubsystem>()->Subscribe(TEXT(">"));
    }

    UFUNCTION()
    void HandleMessage(const FNatsMessage& Message)
    {
        UE_LOG(LogTemp, Warning, TEXT("[NATS SMOKE TEST] %s: %s"), *Message.Subject, *Message.Data);
    }
};
```

Підписка на голий `>` (усе, будь-якої глибини) — навмисно широка, лише для тимчасової
діагностики; для постійного коду завжди підписуйтесь на конкретний subject або вузький
шаблон.

### Автоматизовані тести

Плагін постачається разом із власним набором автотестів на фреймворку Unreal Automation
(`Plugins/NatsClient/Source/NatsClientTests/`) — за тим самим принципом можна писати й
регресійні тести для власної інтеграції.

Запуск через редактор: **Window → Test Automation** (Session Frontend → вкладка
Automation), відфільтрувати за `NatsClient`, обрати потрібні тести, **Start Tests**.

Headless, з командного рядка:

```bash
UnrealEditor-Cmd "<шлях>/YourProject.uproject" \
  -ExecCmds="Automation RunTests NatsClient" \
  -TestExit="Automation Test Queue Empty" \
  -unattended -nopause -nullrhi
```

Частина тестів (усе, що безпосередньо перевіряє розбір протоколу та збірку кадрів) не
потребує зовнішнього сервера. Тести, що перевіряють реальний обмін даними з JetStream,
підключаються до `127.0.0.1:4222` — перед запуском піднімайте локальний сервер, як описано
вище.

---

**Далі:** [13. Поширені запитання](13-FAQ.md)
