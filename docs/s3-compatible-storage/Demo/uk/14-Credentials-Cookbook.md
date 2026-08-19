*[🇬🇧 English](../en/14-Credentials-Cookbook.md) | 🇺🇦 Українська*

[← До змісту](README.md)

# 14. Рецепти облікових даних

Розділ-практикум. Не «де тримати ключі» і не «який ассет завести» — на це вже відповіли
[3. Облікові дані](03-Credentials.md) і [13. Профілі на практиці](13-Profile-Scenarios.md).
Тут — **послідовність дій**: що викликати, у якому порядку, з C++ і з Blueprint, для кожної
з чотирьох конфігурацій.

| Розділ | Відповідає на питання |
|---|---|
| [3. Облікові дані](03-Credentials.md) | Де ключам місце, а де ні, і як плагін їх шукає |
| [13. Профілі на практиці](13-Profile-Scenarios.md) | Які ассети профілів завести під кожну конфігурацію |
| **14 (цей)** | Які саме виклики зробити і в якому порядку |

Кожен сценарій нижче дає пронумеровані кроки, фрагмент C++ і рівноцінний граф Blueprint.

---

## Чотири способи віддати клієнтові ключі

Плагін не має «поля для ключа». Ключі потрапляють до клієнта одним із чотирьох способів, і
вибір між ними — це вибір **хто і коли їх знає**.

| Що у вас на руках | Blueprint | C++ | На що діє |
|---|---|---|---|
| Ключі щойно повернув бекенд, сховище одне | **Set Runtime Credentials** (на підсистемі) | `UDemoS3Subsystem::SetRuntimeCredentials` | Тільки клієнт за замовчуванням |
| Те саме, але сховищ кілька | **Set Static Credentials** (на клієнті) | `UDemoS3Client::SetStaticCredentials` | Той клієнт, на якому викликана — зокрема клієнт профілю |
| Ключі треба брати заново щоразу, коли спливають | — | `UDemoS3Client::SetCredentialsProvider` з `FDemoS3CredentialsProvider_Callback` | Той клієнт |
| Ключі ввів користувач і вони мають пережити перезапуск | **Save S3 Credentials** | `UDemoS3CredentialsLibrary::SaveUserCredentials` | Усі клієнти з джерелом `Local user store` і цим іменем профілю |
| Ключів немає і не буде | `Credential Source = Anonymous` | `FDemoS3CredentialsProvider_Anonymous` | Клієнт |

І зворотні дії:

| Задача | Blueprint | C++ |
|---|---|---|
| Забути ключі, задані кодом | **Clear Runtime Credentials** | `UDemoS3Subsystem::ClearRuntimeCredentials` |
| Забути ключі користувача (один профіль / усі) | **Clear S3 Credentials** / **Clear All S3 Credentials** | `UDemoS3CredentialsLibrary::ClearUserCredentials` / `ClearAllUserCredentials` |
| Змусити клієнта профілю перечитати джерело | **Forget Profile Client** | `UDemoS3Subsystem::ForgetProfileClient` |
| Розв'язати ключі просто зараз, нічого не передаючи | — | `UDemoS3Client::RefreshCredentials` |

`SetCredentialsProvider` і `RefreshCredentials` навмисно не винесені у Blueprint: перша приймає
`TSharedRef` на інтерфейс, друга — `TFunction`. Обидві потрібні рівно там, де вже пишуть код.

> **`Set Static Credentials` — не «те саме, але довше».** Різницю між нею і
> `Set Runtime Credentials` розібрано в
> [13. Профілі на практиці](13-Profile-Scenarios.md#як-профіль-отримує-ключі-чотири-джерела-в-дії):
> друга бачить лише клієнта за замовчуванням, і для клієнта з `Get S3 Client For Profile`
> мовчки нічого не робить.

---

## Яке джерело для якої збірки

`EDemoS3CredentialSource` має рівно чотири значення (`Core/DemoS3Enums.h`). Таблиця нижче — про те,
де кожне доречне, а де воно активно шкідливе.

| Джерело | Гра, яку завантажують гравці | Listen-хост | Виділений сервер | Настільний застосунок | Редактор і CI |
|---|---|---|---|---|---|
| **Environment variables** | ❌ змінних там немає — кожен запит завершиться `Authentication Error`; а якщо ви їх туди покладете, ви віддали ключі гравцеві | ❌ хост — це той самий бінарник, що й у клієнта | ✅ основне призначення | ⚠️ технічно працює, але ключі виходять ваші, а не користувача | ✅ у редакторі підміняється редакторською секцією |
| **Local user store (encrypted)** | ❌ у гравця немає власних ключів S3, і питати їх у нього немає за що | ❌ те саме | ⚠️ сервер запускають без інтерактивного користувача | ✅ основне призначення | ✅ поводиться однаково з боєм |
| **Anonymous** | ✅ для бакета, відкритого лише на читання, і для роботи через підписані посилання | ✅ те саме | ⚠️ серверу зазвичай потрібен запис | ⚠️ лише якщо чуже сховище публічне | ✅ |
| **Supplied in code** | ✅ короткоживучі ключі від вашого бекенда | ✅ те саме | ✅ якщо ключі дає ваш секрет-менеджер, а не оточення | ⚠️ ключі все одно доведеться десь зберегти | ✅ |

Три правила, які випливають із таблиці й не мають винятків:

1. **Усе, що всередині збірки, витягується.** Це не перестраховка — див.
   [«Головне правило»](03-Credentials.md#головне-правило).
2. **`Environment variables` — серверне джерело.** Не «майже підходить» клієнту, а не
   підходить.
3. **`Anonymous` проти приватного бакета дає `Permission Denied`, а не `Authentication Error`.**
   Непідписаний запит з погляду провайдера не «без ключа», а `AccessDenied` — і помилка
   вказує на політику, хоча насправді ви просто не поставили провайдера. Саме тому
   `DemoMakeDefaultCredentialsProvider` навмисно **не** додає `Anonymous` у кінець ланцюжка.

---

## Провайдер, який сам ходить по нові ключі

`FDemoS3CredentialsProvider_Callback` (`Auth/DemoS3CredentialsProviders.h`) — саме те, що перетворює
«мій бекенд видає короткоживучі ключі» на робочу схему. Він робить три речі, кожну з яких
інакше довелося б писати самому:

- **кешує** те, що ви віддали, і не питає вдруге, поки воно чинне;
- **оновлює наперед**, за `RefreshMarginSeconds` секунд до `ExpiresAtUtc` (типово 60), щоб
  запит, підписаний під самий кінець, не прибув уже після завершення строку;
- **склеює одночасні запити**: двадцять передавань, що стартують на протермінованих ключах,
  дадуть вашому бекендові **один** виклик, а не двадцять.

Повний приклад — з поверненням помилки, з терміном життя і з реакцією на відкликаний ключ:

```cpp
#include "DemoS3Subsystem.h"
#include "DemoS3Client.h"
#include "Auth/DemoS3CredentialsProviders.h"

void UMyGameInstance::InstallS3Credentials()
{
    UDemoS3Client* Client = GetSubsystem<UDemoS3Subsystem>()->GetDefaultClient();

    // The fetch function is called on demand: once at the first request, and again shortly
    // before the cached credentials lapse. Never on a timer of your own.
    auto Provider = MakeShared<FDemoS3CredentialsProvider_Callback, ESPMode::ThreadSafe>(
        FDemoS3CredentialsProvider_Callback::FDemoS3CredentialsFetch::CreateWeakLambda(this,
            [this](FDemoS3CredentialsResolved OnFetched)
            {
                MyBackend::RequestS3Credentials(
                    [OnFetched](bool bOk, const FStsResponse& Response)
                    {
                        if (!bOk)
                        {
                            // Answer anyway. The delegate must fire exactly once per call -
                            // swallowing a failure leaves every waiting transfer hanging.
                            OnFetched.ExecuteIfBound(false, FDemoS3Credentials());
                            return;
                        }

                        // All three values, not two: a temporary key without its session
                        // token is rejected as though it did not exist.
                        OnFetched.ExecuteIfBound(true, FDemoS3Credentials(
                            Response.AccessKeyId,
                            Response.SecretAccessKey,
                            Response.SessionToken,
                            Response.ExpiresAtUtc));   // UTC, and it is what drives the refresh
                    });
            }),
        /*InRefreshMarginSeconds=*/90.f);   // raise it if your backend is slow to answer

    Client->SetCredentialsProvider(Provider);
}
```

**Чого робити не треба.** Не заводьте власний таймер оновлення й не викликайте
`RefreshCredentials` за розкладом: провайдер уже питає рівно тоді, коли потрібно, а зайвий
виклик — це зайвий запит до вашого бекенда.

**Відкликаний ключ.** Якщо провайдер відповів `SignatureDoesNotMatch`, `InvalidAccessKeyId`,
`ExpiredToken`, `InvalidToken`, `TokenRefreshRequired` або `RequestTimeTooSkewed`, плагін сам
викликає `Invalidate()` на провайдері, і наступний запит піде по свіжі ключі. Звичайний
`AccessDenied` цього **не** робить навмисно: ключ правильний, йому бракує прав, і повторне
питання дало б той самий ключ, тільки навантаживши бекенд.

**Що з цього доступно в Blueprint.** Нічого — і це не обмеження, а межа можливого: параметр
провайдера — це функція, а не значення. Blueprint-еквівалент callback-провайдера — викликати
**Set Runtime Credentials** (або **Set Static Credentials**) щоразу, коли ваш бекенд відповів,
із заповненим `Expires In Seconds`:

```
On Backend Auth Response
   │
   └─► Get S3 Subsystem → Set Runtime Credentials
           Access Key Id      : (від бекенда)
           Secret Access Key  : (від бекенда)
           Session Token      : (від бекенда)
           Expires In Seconds : 3600
```

Різниця в одному: тут **ви** відповідаєте за те, щоб покликати бекенд знову до закінчення
строку. Провайдер зробив би це сам.

---

## Сценарій 1. Однокористувацька гра: «п'ятнашки» з фоном зі сховища

Гра — класичні п'ятнашки. Фонове зображення, яке розрізається на 15 плиток, не лежить у
збірці: воно береться зі сховища, тож набір картинок можна поповнювати без патча.

### Кроки

1. Покладіть у бакет **звичайний файл зображення** — `puzzle/backgrounds/city.jpg`, з
   `Content-Type: image/jpeg`.
2. Зчитайте його **в пам'ять**, а не у файл: `S3 Download Bytes` / `UDemoS3Client::DownloadBytes`.
   Проміжний файл тут ні до чого — байти одразу підуть у декодер.
3. Зберіть із байтів `UTexture2D` через `FImageUtils::ImportBufferAsTexture2D`
   (`ImageUtils.h`, модуль `Engine`). У Blueprint те саме робить нода
   **Import Buffer as Texture 2D** (категорія `Rendering`).
4. **Збережіть посилання на текстуру в `UPROPERTY`** — вона транзієнтна, пакета в неї немає,
   і без посилання її забере збирач сміття, залишивши вам чорну плитку.
5. Використайте її: `Set Texture Parameter Value` на динамічному екземплярі матеріалу дошки
   або `Set Brush from Texture` на віджеті `Image`.

### C++

```cpp
// PuzzleBoard.h
UCLASS()
class APuzzleBoard : public AActor
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;

private:
    void OnBackgroundDownloaded(const FDemoS3OperationResult& Result, const TArray<uint8>& Data);

    /** Keeps the runtime texture alive: it belongs to no package, so nothing else holds it. */
    UPROPERTY(Transient)
    TObjectPtr<UTexture2D> BackgroundTexture;

    UPROPERTY(Transient)
    TObjectPtr<UMaterialInstanceDynamic> BoardMaterial;
};
```

```cpp
// PuzzleBoard.cpp
#include "ImageUtils.h"     // FImageUtils::ImportBufferAsTexture2D
#include "DemoS3Subsystem.h"
#include "DemoS3Client.h"

void APuzzleBoard::BeginPlay()
{
    Super::BeginPlay();

    UDemoS3Client* Client = GetGameInstance()->GetSubsystem<UDemoS3Subsystem>()->GetDefaultClient();

    // Straight into memory: the bytes go to a decoder, never to disk.
    // CreateUObject is safe here - the plugin invokes the delegate with ExecuteIfBound, so
    // an actor destroyed mid-download simply stops being called.
    Client->DownloadBytes(
        TEXT("my-game-content"),
        TEXT("puzzle/backgrounds/city.jpg"),
        FDemoS3OnDownloadResult::CreateUObject(this, &APuzzleBoard::OnBackgroundDownloaded));
}

void APuzzleBoard::OnBackgroundDownloaded(const FDemoS3OperationResult& Result, const TArray<uint8>& Data)
{
    if (!Result.IsSuccess())
    {
        // ToLogString carries status, code, message and the hint about what to change.
        UE_LOG(LogTemp, Warning, TEXT("Background unavailable, using the built-in one: %s"),
            *Result.ToLogString());
        return;
    }

    // Decodes PNG, JPEG, BMP, TGA, EXR, HDR, TIFF - whatever the engine's ImageWrapper
    // module recognises. Returns null when the bytes are not an image at all.
    BackgroundTexture = FImageUtils::ImportBufferAsTexture2D(Data);
    if (!BackgroundTexture)
    {
        UE_LOG(LogTemp, Warning, TEXT("Downloaded %d bytes, but they are not a picture."), Data.Num());
        return;
    }

    BoardMaterial->SetTextureParameterValue(TEXT("Background"), BackgroundTexture);
}
```

Три властивості цього шляху, про які краще знати заздалегідь:

- **Текстура транзієнтна й має один рівень мипів.** Для UI та для плоскої дошки п'ятнашок
  це саме те, що треба; для об'єкта, який видно здалеку, мипів бракуватиме, і він
  «шумітиме».
- **Декодування синхронне.** Плагін гарантує, що зворотний виклик приходить у ігровий потік
  (див. [6. C++ API](06-Cpp-API.md)), тож розпакування 4K-JPEG — це помітний ривок кадру.
  Тримайте у бакеті зображення того розміру, який справді потрібен, або зберігайте кілька
  розмірів і зчитуйте потрібний.
- **`Content-Type` в об'єкті ні на що тут не впливає** — формат визначається за самими
  байтами. Проставляти його все одно варто: за ним об'єкт відкриється у браузері й у консолі
  провайдера.

### Blueprint

```
Event BeginPlay
   │
   └─► Get S3 Subsystem → Get Default S3 Client
          │
          └─► S3 Download Bytes
                 Bucket Name : my-game-content
                 Object Key  : puzzle/backgrounds/city.jpg
                    │
                    ├─ On Success → Data ──► Import Buffer as Texture 2D
                    │                            │
                    │                            └─► Return Value
                    │                                   │
                    │                                   ├─► SET Background Texture   ← змінна актора;
                    │                                   │                              без неї текстуру забере GC
                    │                                   │
                    │                                   └─► Set Texture Parameter Value
                    │                                          Target         : Board Material
                    │                                          Parameter Name : Background
                    │                                          Value          : Return Value
                    │
                    └─ On Failure → Result → Get S3 Diagnostic Hint → лишити вбудований фон
```

Змінна `Background Texture` тут не «щоб було зручно» — це і є те, що тримає текстуру живою.
Якщо покласти `Import Buffer as Texture 2D` одразу в `Set Texture Parameter Value` і нікуди
не зберігати, картинка зникне на найближчому проході збирача сміття.

### Те саме для звуку: клацання плитки зі сховища

Разом із фоном природно тягнути й звукову тему набору — клацання плитки, звук перемоги.
Схема та сама: байти в пам'ять, збірка об'єкта в рантаймі, посилання в `UPROPERTY`. Різниця
лише в тому, чим саме збирати.

Покладіть у бакет **16-бітний PCM `.wav`** — `puzzle/sfx/tile-move.wav`, `Content-Type:
audio/wav`. Формат тут не примха: шлях нижче читає заголовок WAV і віддає рушієві готовий
PCM. MP3 і OGG так не заходять — їхні декодери в рушії прив'язані до кукнутих ассетів, тож
для стиснених форматів вам знадобиться власний кодек.

```cpp
#include "Audio.h"                       // FWaveModInfo
#include "Sound/SoundWaveProcedural.h"
#include "Kismet/GameplayStatics.h"

// Same UPROPERTY rule as the texture: a runtime-built sound has no package, so without a
// hard reference the collector takes it and the tile clicks in silence.
UPROPERTY(Transient)
TObjectPtr<USoundWaveProcedural> TileClickSound;

void APuzzleBoard::FetchTileClickSound(UDemoS3Client* Client)
{
    TWeakObjectPtr<APuzzleBoard> WeakThis(this);

    Client->DownloadBytes(
        TEXT("puzzle-assets"), TEXT("puzzle/sfx/tile-move.wav"),
        FDemoS3OnDownloadResult::CreateLambda(
            [WeakThis](const FDemoS3OperationResult& Result, const TArray<uint8>& Bytes)
            {
                if (!WeakThis.IsValid() || !Result.IsSuccess())
                {
                    return;
                }

                // Parses the RIFF header in place: WaveInfo points into Bytes rather than
                // copying, so it stays valid only for as long as Bytes does - which is this
                // callback. QueueAudio below takes its own copy, so that is where it ends.
                FWaveModInfo WaveInfo;
                if (!WaveInfo.ReadWaveInfo(Bytes.GetData(), Bytes.Num()))
                {
                    return;   // not a WAV this path can read
                }

                USoundWaveProcedural* Sound = NewObject<USoundWaveProcedural>();
                Sound->SetSampleRate(*WaveInfo.pSamplesPerSec);
                Sound->NumChannels = *WaveInfo.pChannels;
                Sound->SoundGroup  = SOUNDGROUP_Effects;
                Sound->bLooping    = false;

                const int32 BytesPerFrame = *WaveInfo.pChannels * (*WaveInfo.pBitsPerSample / 8);
                Sound->Duration = BytesPerFrame > 0
                    ? (float)WaveInfo.SampleDataSize / (BytesPerFrame * *WaveInfo.pSamplesPerSec)
                    : 0.f;

                // Copies the samples into the queue the audio thread drains.
                Sound->QueueAudio(WaveInfo.SampleDataStart, WaveInfo.SampleDataSize);

                WeakThis->TileClickSound = Sound;
            }));
}
```

Далі — звичайний `UGameplayStatics::PlaySound2D(this, TileClickSound)`.

> **Одна відмінність від текстури, про яку варто знати заздалегідь.**
> `USoundWaveProcedural` — це **черга**, а не буфер: програвання її вичерпує. Для звуку, який
> лунає раз за гру, цього досить; для клацання плитки, яке звучить сотні разів, тримайте
> завантажені байти в `TArray<uint8>` і викликайте `QueueAudio` перед кожним відтворенням
> (або зберіть свій `USoundWave` із `RawPCMData`, якщо потрібне повноцінне повторне
> використання).

У Blueprint готової ноди для збирання звуку з байтів немає — на відміну від
`Import Buffer as Texture 2D`. Тобто цей крок доведеться загорнути у власну
`UFUNCTION(BlueprintCallable)`, а вже її кликати з графа рівно там, де у прикладі з фоном
стоїть `Import Buffer as Texture 2D`:

```
S3 Download Bytes
   Bucket Name : puzzle-assets
   Object Key  : puzzle/sfx/tile-move.wav
   │
   ├─ On Success → Data ──► Make Sound From Wav Bytes   ← ваша UFUNCTION з коду вище
   │                            │
   │                            └─► Return Value → Set (змінна Tile Click Sound)
   │
   └─ On Failure → Result → Get S3 Diagnostic Hint → лишити вбудований звук
```

Для облікових даних тут не змінюється **нічого**: це той самий бакет, той самий клієнт і те
саме питання «що буде, якщо ключ витече» — просто інший об'єкт у ньому.

### І тепер найголовніше: звідки в цій грі беруться ключі

Гра однокористувацька — але вона **їде до гравців**, а отже підпадає під
[«Головне правило»](03-Credentials.md#головне-правило) цілком: усе, що є в збірці, звідти
дістануть. Конкретно для нашого фону це означає ось що.

| Варіант | Що в збірці | Чого коштує | Коли брати |
|---|---|---|---|
| **Публічний бакет + `Anonymous`** | нічого | Картинки може завантажити будь-хто, і трафік у рахунку ваш | Фон, патчі, будь-який вміст лише на читання — **це нормальна відповідь**, а не компроміс |
| **Підписані посилання від вашого бекенда** | нічого | Потрібен бекенд і мережа — офлайнова гра стає онлайновою | Коли вміст платний або персональний |
| **Короткоживучі ключі від вашого бекенда** | нічого | Те саме плюс STS на боці бекенда | Коли грі, крім читання, потрібен ще й запис — хмарні збереження |
| **Вшитий ключ лише на читання** | ключ | Витік коштує трафіку, не даних | Коли бекенда немає, а бакет публічним робити не можна |
| ~~`Environment variables`~~ | нічого | Кожен запит — `Authentication Error` | Ніколи в клієнтській збірці |

**Публічний бакет на читання — легітимна відповідь.** Фон для п'ятнашок не є секретом; він
однаково опиниться на диску гравця через п'ять секунд після старту гри. Єдине, що ви цим
віддаєте — можливість завантажувати ваші картинки без гри, і платите за це трафіком. Ставте
перед бакетом CDN, тримайте в ньому **лише** вміст лише для читання, і питання закрито.

Перевірка на одне питання, з розділу [3](03-Credentials.md#c-публічний-контент-лише-для-читання):
**що станеться, якщо цей ключ завтра з'явиться на форумі?** Для ключа лише на читання одного
публічного бакета відповідь — «нічого», і тоді вбудовувати можна.

Щойно в тій самій грі з'являються хмарні збереження — це вже **друге** сховище з іншими
правами й іншим джерелом ключів, а не те саме. Розкладку на два ассети профілів дає
[13. Профілі на практиці](13-Profile-Scenarios.md#сценарій-1-однокористувацька-гра).

---

## Сценарій 2. Мережева гра: listen-сервер

Хост — водночас сервер і гравець, і його бінарник нічим не відрізняється від клієнтського.
Тому довгоживучих ключів тут не буває **ні в кого**, включно з хостом.

### Варіант A. Хост робить усе

Стан світу належить сесії, а не гравцеві. Ключі отримує тільки хост, і тільки він звертається
до сховища.

**Кроки:**

1. Заведіть профіль з `Credential Source = Supplied in code` (наприклад `SP_WorldState`).
2. Після автентифікації **на боці хоста** попросіть у бекенда ключі з політикою на префікс
   сесії — `sessions/${session_id}/*`.
3. Віддайте їх клієнтові **саме цього профілю** через `SetStaticCredentials`.
4. Кожну операцію закривайте перевіркою повноважень, інакше подія виконається у всіх.

```cpp
void AMySessionState::OnSessionCredentialsReceived(const FStsResponse& Response)
{
    // Authority only: on a listen server this code path exists on every machine.
    if (!HasAuthority())
    {
        return;
    }

    UDemoS3Subsystem* Subsystem = GetGameInstance()->GetSubsystem<UDemoS3Subsystem>();
    UDemoS3Client*    Client    = Subsystem->GetClientForProfile(WorldStateProfile);

    // Applies to this client only - unlike SetRuntimeCredentials, which the subsystem
    // applies to the default client and nothing else.
    Client->SetStaticCredentials(
        Response.AccessKeyId,
        Response.SecretAccessKey,
        Response.SessionToken,
        /*ExpiresInSeconds=*/3600);
}
```

```
Event On Session Credentials Received
   │
   └─► Switch Has Authority
          ├─ Authority → Get S3 Subsystem → Get S3 Client For Profile (SP_WorldState)
          │                 │
          │                 └─► Set Static Credentials
          │                        Access Key Id      : (від бекенда)
          │                        Secret Access Key  : (від бекенда)
          │                        Session Token      : (від бекенда)
          │                        Expires In Seconds : 3600
          │
          └─ Remote → нічого не робити
```

Далі — сама операція, теж під `Switch Has Authority`:

```
Event Save World State
   │
   └─► Switch Has Authority
          ├─ Authority → Get S3 Client For Profile (SP_WorldState) → S3 Upload File
          │                 ├─ On Success → Multicast: «світ збережено»
          │                 └─ On Failure → Result → Get S3 Diagnostic Hint
          └─ Remote    → нічого не робити
```

Без цієї перевірки сесія на вісьмох дасть вісім однакових завантажень одного файлу, вісім
оплачених запитів і непередбачуваний порядок запису.

### Варіант B. У кожного клієнта свої короткоживучі ключі

Виправдано, коли дані належать конкретному гравцеві: його скріншоти, його профіль, його
налаштування. Тут кожен клієнт ходить до сховища сам, **своїми** ключами.

**Кроки:**

1. Політика ключа обмежена префіксом цього гравця — `users/${user_id}/*`. Спільний ключ на
   всіх означає, що будь-хто може перезаписати чужі дані.
2. Ключі бере **кожен процес окремо**: `UDemoS3Subsystem` — це `UGameInstanceSubsystem`, у хоста
   своя, у кожного клієнта своя, і нічого між ними не реплікується
   (див. [10. Сценарії розгортання](10-Deployment.md#підсистема-і-мережа)).
3. Сесія довша за строк життя ключів — тому тут доречний саме
   [callback-провайдер](#провайдер-який-сам-ходить-по-нові-ключі), а не одноразовий виклик:
   він оновить ключі посеред матчу без жодного рядка коду з вашого боку.
4. Ключі **не передавайте по мережі**. RPC із ключем у параметрі — це ключ у трафіку й у
   логах; кожен клієнт має отримати свої від бекенда напряму.

```cpp
// On each client, after it has authenticated against your backend itself.
UDemoS3Client* Client = GetGameInstance()->GetSubsystem<UDemoS3Subsystem>()->GetDefaultClient();

Client->SetCredentialsProvider(
    MakeShared<FDemoS3CredentialsProvider_Callback, ESPMode::ThreadSafe>(
        FDemoS3CredentialsProvider_Callback::FDemoS3CredentialsFetch::CreateWeakLambda(this,
            [this](FDemoS3CredentialsResolved OnFetched)
            {
                // Scoped to this player's own prefix by the backend, not by the client.
                MyBackend::RequestS3CredentialsForCurrentPlayer(OnFetched);
            })));
```

```
Event On Player Authenticated        (виконується на кожному клієнті окремо)
   │
   └─► Get S3 Subsystem → Set Runtime Credentials
           Access Key Id      : (від бекенда, для ЦЬОГО гравця)
           Secret Access Key  : (від бекенда)
           Session Token      : (від бекенда)
           Expires In Seconds : 900
   │
   └─► ... далі S3 Upload File з ключем users/{PlayerId}/screenshot.png
```

Із Blueprint строк доведеться оновлювати самотужки: покличте бекенд повторно й викличте
**Set Runtime Credentials** ще раз до того, як `Expires In Seconds` спливе.

---

## Сценарій 3. Виділений сервер

Єдина конфігурація, де довгоживучі ключі доречні: бінарник сервера у гравців не буває.

### Кроки: змінні оточення

1. `Credential Source = Environment variables` — у Project Settings або на ассеті профілю.
2. Якщо сховищ кілька, дайте кожному профілю власний **Environment Variable Prefix**. Поле
   є **тільки на ассеті профілю**, у Project Settings його немає: спільна конфігурація
   завжди читає стандартний `AWS`.
3. Проставте змінні у тому, чим ви запускаєте процес: systemd unit, Docker, скрипт,
   роль інстанса, менеджер секретів.
4. Нічого не викликайте з коду. Це і є сенс цього джерела.

```bash
# Standard names: the same ones aws, mc, rclone and the rest read.
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...        # temporary credentials only

# A second storage, on the profile whose prefix is REPLAYS.
export REPLAYS_ACCESS_KEY_ID=...
export REPLAYS_SECRET_ACCESS_KEY=...

./MyGameServer -log
```

Значення читаються **перед кожним запитом**, тож ротація ключів діє без перезапуску процесу.

Перевірити, що сервер справді їх бачить, найдешевше консоллю розробника — модуль
`S3CompatibleStorageDemoDebug` присутній і в зібраній Development-збірці:

```
> DemoS3TestConnection
```

Повний перелік команд — у [12. Консоль розробника](12-Debug-Console.md).

### Клієнт без ключів узагалі: підписані посилання

Схема, за якої на клієнті немає **нічого**, що можна видобути.

**Кроки:**

1. Клієнт налаштований як `Anonymous`.
2. Клієнт надсилає серверу RPC: «хочу залити збереження».
3. Сервер перевіряє право — це і є та єдина точка, де рішення ухвалюється, — і підписує
   посилання.
4. Сервер повертає рядок клієнтові.
5. Клієнт робить звичайний HTTP-запит **тим самим методом**, яким посилання підписане.

```cpp
// Server side.
const FDemoS3PresignedUrlResult Url = Client->GeneratePresignedUrl(
    TEXT("my-game-saves"),
    FString::Printf(TEXT("saves/%s/slot1.sav"), *PlayerId),
    EDemoS3HttpMethod::PUT,
    /*ExpiresInSeconds=*/900);   // fifteen minutes is plenty for one upload

if (Url.IsSuccess())
{
    // Url.ExpiresAt is already computed: UtcNow plus the lifetime, so the client can show
    // "valid for another N minutes" without subtracting anything itself.
    ClientReceiveUploadUrl(Url.Url, Url.ExpiresAt);
}
```

```
Клієнт: RPC до сервера — «хочу залити збереження»
   │
Сервер (Switch Has Authority → Authority)
   │
   └─► Get S3 Subsystem → Get Default S3 Client
          │
          └─► Make Presigned URL                      ← чиста (pure) нода, без пінів виконання
                 Bucket Name        : my-game-saves
                 Object Key         : saves/{PlayerId}/slot1.sav
                 Method             : PUT
                 Expires In Seconds : 900
                 │
                 └─► Return Value ──► Client RPC: Receive Upload Url
                                          │
                                    Клієнт: звичайний HTTP PUT за цією адресою
```

Що варто пам'ятати про самі посилання:

- Посилання дійсне **лише для свого методу**. PUT-посилання, відкрите у браузері, не
  спрацює: браузер надсилає GET.
- Максимум за специфікацією Signature Version 4 — 604800 секунд (7 діб). Більше значення
  плагін не відхиляє, а мовчки обрізає до максимуму й пише попередження в лог.
- Посилання дає доступ **будь-кому**, хто його має. Хвилини, не доба.
- Підпис рахується локально, без звернення до провайдера, — тому ноди **Make Presigned URL**
  і **Generate Presigned URL** чисті й не мають `On Success`. Різницю між ними розібрано в
  [4. Операції у Blueprint](04-Blueprint-Operations.md#яку-з-двох-нод-брати).

### Засідка: підписане посилання на callback-провайдері

Підпис **синхронний**, а `FDemoS3CredentialsProvider_Callback` може відповідати асинхронно. Якщо
на момент виклику `GeneratePresignedUrl` у провайдера ще нічого не закешовано, він не встигне
відповісти, і результат буде `Authentication Error` із порожнім `Url` — при цілком робочих ключах.

`Static`, `Environment` і `LocalUserStore` відповідають одразу, тож їх це не стосується
взагалі. Для callback-провайдера один раз розігрійте кеш:

```cpp
// Once, before the first presigned URL - not before every one: the provider caches.
Client->RefreshCredentials([this, Client, PlayerId](bool bSuccess)
{
    if (!bSuccess)
    {
        return;
    }

    const FDemoS3PresignedUrlResult Url = Client->GeneratePresignedUrl(
        TEXT("my-game-saves"),
        FString::Printf(TEXT("saves/%s/slot1.sav"), *PlayerId),
        EDemoS3HttpMethod::PUT, 900);

    ClientReceiveUploadUrl(Url.Url, Url.ExpiresAt);
});
```

Найпростіше цього уникнути інакше: на виділеному сервері беріть `Environment variables`. Тоді
питання не виникає взагалі.

> **Не кладіть серверний профіль у клієнтську збірку** в надії, що «там просто не спрацює».
> Воно справді не спрацює — але помилка виглядатиме як `Authentication Error` без очевидної причини.

---

## Сценарій 4. Настільний застосунок

Бакет належить **користувачеві**. Ключі — його, а не ваші: їх не можна ані вшити, ані
вимагати для них бекенд.

**Ассет:** `Credential Source = Local user store (encrypted)`, `Local Store Profile Name`
на ваш розсуд (типово `default`).

### Крок 1. Екран введення

Ваш віджет налаштувань має три поля вводу й кнопку. Кнопка викликає **Save S3 Credentials**.

```
(натиснуто «Підключитися»)
   │
   └─► Save S3 Credentials
          Profile Name      : default
          Access Key Id     : (з поля вводу)
          Secret Access Key : (з поля вводу)
          Session Token     : (порожньо, якщо ключі не тимчасові)
             │
             ├─ Return Value = False → тека налаштувань користувача недоступна для запису;
             │                          покажіть це, бо мовчазна невдача тут — найгірше
             └─ Return Value = True  → далі до перевірки
```

```cpp
#include "Blueprint/DemoS3CredentialsLibrary.h"

const bool bSaved = UDemoS3CredentialsLibrary::SaveUserCredentials(
    TEXT("default"),
    AccessKeyIdField->GetText().ToString(),
    SecretAccessKeyField->GetText().ToString(),
    /*SessionToken=*/FString());

if (!bSaved)
{
    // The only reason this returns false: the user's settings directory is not writable.
    ShowError(NSLOCTEXT("MyApp", "SaveFailed", "Не вдалося зберегти ключі."));
    return;
}
```

`Session Token` у ноді складений під `AdvancedDisplay` — його видно після розкриття стрілки
внизу ноди. Заповнюйте лише для тимчасових ключів: тимчасовий ключ без свого токена
провайдер відхилить так, ніби такого ключа не існує.

### Крок 2. Скинути кеш і перевірити — одразу

Це той крок, який економить найбільше підтримки. Та сама помилка через півгодини, під час
першого справжнього збереження, вже нічого користувачеві не пояснює.

```
Save S3 Credentials (успішно)
   │
   └─► Forget Profile Client (SP_UserStorage)      ← клієнт кешує вже розв'язані ключі
          │
          └─► Get S3 Client For Profile (SP_UserStorage) → S3 List Objects
                 Bucket Name : (з поля вводу користувача)
                 Prefix      : (порожньо)
                 Max Keys    : 1
                    ├─ On Success → до основного екрана
                    └─ On Failure → Result → Get S3 Diagnostic Hint → показати текст поруч із полями
```

**`Forget Profile Client` тут обов'язковий.** Ключі змінилися **за** джерелом, а не через
ноду: клієнт, який вже колись їх розв'язав, продовжить користуватися старими, і перевірка
покаже стару відмову. Порівняйте зі `Set Static Credentials`, після якої `Forget Profile
Client` не потрібен — вона замінює ключі на клієнті одразу.

`S3 List Objects` з `Max Keys = 1` — найдешевша перевірка, яка водночас підтверджує адресу,
регіон, ключі та існування бакета. Один запит, одна відповідь. Піни `Max Keys` і
`Continuation Token` складені під `AdvancedDisplay`: розкрийте стрілку внизу ноди.

### Крок 3. Старт застосунку

```
Event Init
   │
   └─► Has S3 Credentials (default)
          ├─ True  → одразу до основного екрана
          └─ False → показати екран підключення
```

`Has S3 Credentials` перевіряє наявність, **не розшифровуючи** файл, — тож це дешево і його
можна кликати на кожному старті.

### Крок 4. Вихід і «забути мене»

| Дія | Нода | C++ |
|---|---|---|
| Вийти з одного акаунта | **Clear S3 Credentials** (`default`) | `UDemoS3CredentialsLibrary::ClearUserCredentials` |
| Стерти все | **Clear All S3 Credentials** | `UDemoS3CredentialsLibrary::ClearAllUserCredentials` |
| Показати, де лежить файл | **Get S3 Credentials File Path** | `UDemoS3CredentialsLibrary::GetCredentialsFilePath` |

Після будь-якого з двох очищень викличте **Forget Profile Client** — з тієї самої причини, що
й у кроці 2.

**Get S3 Credentials File Path** варто винести в інтерфейс кнопкою «показати теку»: це
чесна відповідь користувачеві на питання «а що ви про мене зберігаєте і як це видалити
руками».

### Крок 5. Кілька акаунтів або кілька застосунків

- **Кілька акаунтів в одному застосунку** — своє `Local Store Profile Name` на кожен, і по
  ассету профілю на кожен. Порожні (однакові) імена означають, що обидва читають одні й ті
  самі ключі.
- **Кілька ваших застосунків на одній машині** — робити нічого не треба: сховище солиться
  назвою проєкту (`FApp::GetProjectName()`), тож два різні застосунки не прочитають ключі
  один одного навіть за однакового імені профілю.

### Що це шифрування дає, а чого — ні

Файл лежить під теками налаштувань поточного користувача, поза проєктом, зашифрований
AES-256, ключ виведено з ідентифікатора машини та назви застосунку. Це захищає від читання
очима, від витоку через резервну копію й від перенесення файлу на іншу машину — і **не**
захищає від того, хто вже виконує код на цій машині від імені цього користувача. Докладно і
без прикрас — у [3. Облікові дані](03-Credentials.md#як-саме-це-зберігається).

Якщо потрібні системні гарантії — Keychain, DPAPI, libsecret, — реалізуйте
`IDemoS3CredentialsStore` (`Auth/IDemoS3CredentialsStore.h`) поверх них і поставте свою реалізацію в
`FDemoS3CredentialsProvider_LocalUserStore`. Решта плагіна при цьому не змінюється.

---

## Довідник: де що лежить

| Що | Заголовок |
|---|---|
| `EDemoS3CredentialSource` — чотири джерела | `Core/DemoS3Enums.h` |
| `FDemoS3Credentials`, `IsTemporary`, `NeedsRefresh` | `Auth/DemoS3Credentials.h` |
| `IDemoS3CredentialsProvider`, `FDemoS3CredentialsResolved` | `Auth/IDemoS3CredentialsProvider.h` |
| `_Static` · `_Anonymous` · `_Environment` · `_LocalUserStore` · `_Callback` · `FDemoS3CredentialsProviderChain` | `Auth/DemoS3CredentialsProviders.h` |
| `DemoMakeDefaultCredentialsProvider`, `DemoMakeCredentialsProviderForSource` | `Auth/DemoS3CredentialsProviders.h` |
| `IDemoS3CredentialsStore`, `FDemoS3LocalCredentialsStore` | `Auth/IDemoS3CredentialsStore.h` |
| `SaveUserCredentials` та решта нод користувацького сховища | `Blueprint/DemoS3CredentialsLibrary.h` |
| `SetRuntimeCredentials`, `ClearRuntimeCredentials`, `ForgetProfileClient` | `DemoS3Subsystem.h` |
| `SetStaticCredentials`, `SetCredentialsProvider`, `RefreshCredentials`, `GeneratePresignedUrl` | `DemoS3Client.h` |
| `MakePresignedUrl` | `DemoS3BlueprintLibrary.h` |

`DemoMakeCredentialsProviderForSource` — єдине місце, яке перетворює «звідки брати ключі» на
справжнього провайдера. І сторінка налаштувань, і кожен ассет профілю ходять саме через неї,
тому поводяться однаково; раніше в кожної була своя копія цього рішення, і вони встигли
розійтися.

---

## Коли не працює

| Симптом | Найімовірніша причина | Що зробити |
|---|---|---|
| `Authentication Error` у зібраній грі, у редакторі все добре | `Environment variables` плюс редакторська секція, якої у збірці немає | Змініть джерело: `Anonymous`, `Supplied in code` або `Local user store` |
| `Permission Denied` там, де очікували `Authentication Error` | Джерело `Anonymous`, а бакет приватний: непідписаний запит для провайдера — це `AccessDenied` | Поставте провайдера або відкрийте бакет на читання свідомо |
| Ключі задали, профіль їх не бачить | Викликали `Set Runtime Credentials` замість `Set Static Credentials` | Перша діє лише на клієнта за замовчуванням |
| Ключі змінили, працюють старі | Клієнт кешує розв'язані ключі | **Forget Profile Client**. Після `Set Static Credentials` не потрібно |
| Порожній рядок замість підписаного посилання | Callback-провайдер ще нічого не закешував, а підпис синхронний | Один раз `RefreshCredentials` перед першим посиланням |
| Тимчасові ключі відхиляються, хоча ключ і секрет правильні | Не передали `Session Token` | Передавайте всі три значення |
| Бекенд отримує сплеск запитів за ключами | Ключі ставлять вручну на кожну операцію | Поставте `FDemoS3CredentialsProvider_Callback` — він склеює одночасні запити в один |
| Текстура або звук з'явилися й через кілька секунд зникли | На них ніхто не посилається, їх забрав GC | `UPROPERTY` у C++ або змінна в Blueprint |
| `Read Wave Info` повертає `false` на цілком робочому файлі | Це не 16-бітний PCM `.wav` — MP3 чи OGG цей шлях не декодує | Кладіть у бакет `.wav`, або декодуйте своїм кодеком |

Перший рядок звіту Test Connection завжди називає джерело, з якого прийшли ключі, —
`Environment`, `LocalUserStore(<ім'я>)`, `Static(editor keys for <профіль>)`,
`Static(editor settings)`, `Static(supplied in code)`, `Anonymous`. Коли незрозуміло, «яким
саме ключем це підписали», починати треба звідти:
[3. Облікові дані](03-Credentials.md#як-плагін-шукає-ключі-під-час-роботи).

---

[← До змісту](README.md)
