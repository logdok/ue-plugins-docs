*🇺🇦 Українська*

[← До змісту](README.md)

# 6. C++ API

Той самий функціонал, що й у Blueprint, без проміжного шару.

---

## Підключення модуля

```csharp
// YourModule.Build.cs
PrivateDependencyModuleNames.Add("S3CompatibleStorage");
```

Заголовки, які знадобляться:

```cpp
#include "S3Subsystem.h"   // отримати готового клієнта
#include "S3Client.h"      // операції
#include "S3Types.h"       // структури результатів і конфігурації
```

Окремого «збірного» заголовка немає навмисно: явні включення не подовжують час компіляції
всім, хто користується плагіном.

---

## Отримання клієнта

```cpp
US3Client* Client = GetGameInstance()->GetSubsystem<US3Subsystem>()->GetDefaultClient();
```

Підсистема бере конфігурацію з налаштувань проєкту й тримає клієнта живим.

Якщо конфігурація складається в коді:

```cpp
FS3Config Config;
Config.Endpoint.Provider    = ES3Provider::CloudflareR2;
Config.Endpoint.EndpointURL = TEXT("https://<account>.r2.cloudflarestorage.com");
Config.Endpoint.Region      = TEXT("auto");
Config.Normalize();

US3Client* Client = US3Client::CreateClient(Config);
```

> **Час життя.** `US3Client` — це `UObject`, тож на нього має хтось посилатися: `UPROPERTY`
> або `TStrongObjectPtr`. Клієнт із підсистеми про це вже подбав.
>
> Операції, що вже виконуються, тримають себе живими самостійно, тож збирання клієнта
> посеред передавання нічого не зламає — передавання просто дійде до кінця.

---

## Завантаження

```cpp
FS3TransferHandleRef Transfer = Client->UploadFile(
    TEXT("my-bucket"),
    TEXT("saves/player.sav"),
    LocalPath,
    TEXT("application/octet-stream"),
    {},                                   // метадані
    FS3OnUploadResult::CreateLambda(
        [](const FS3UploadResult& Result)
        {
            if (Result.IsSuccess())
            {
                UE_LOG(LogTemp, Log, TEXT("Завантажено, ETag %s, %lld байт%s"),
                    *Result.ETag, Result.BytesUploaded,
                    Result.bWasMultipart ? TEXT(", багаточастинно") : TEXT(""));
            }
            else
            {
                // ToLogString містить статус, код, повідомлення і підказку.
                UE_LOG(LogTemp, Error, TEXT("%s"), *Result.ToLogString());
            }
        }),
    FS3OnProgress::CreateLambda(
        [](const FS3TransferProgress& Progress)
        {
            if (Progress.bTotalKnown)
            {
                UE_LOG(LogTemp, Verbose, TEXT("%.1f%%"), Progress.Percentage);
            }
        }));
```

Із пам'яті — `UploadBytes` з тими самими параметрами, але масивом замість шляху.

**Про `LocalPath`.** Приймається і абсолютний шлях (`"C:/Users/You/save.png"`,
`"/Users/You/save.png"`), і відносний кореня проєкту (`"Saved/SaveGames/player.sav"`) —
другий розгортається через
`FPaths::ConvertRelativePathToFull(FPaths::ProjectDir(), LocalPath)`, тож той самий рядок
веде в те саме місце і в редакторі, і в зібраній грі, на будь-якій платформі. `DownloadFile`
і `DownloadFileChunked` приймають шлях призначення так само. Це саме той механізм, що стоїть
за нодою `Local File Path` у Blueprint — див.
[«Шлях до файлу»](04-Blueprint-Operations.md#шлях-до-файлу-абсолютний-чи-відносний) для
розгорнутих прикладів під кожну платформу.

---

## Зчитування

```cpp
// У файл: стримиться на диск, пам'ять не залежить від розміру.
Client->DownloadFile(TEXT("my-bucket"), TEXT("patches/1.2.pak"), LocalPath,
    FS3OnDownloadResult::CreateLambda(
        [](const FS3OperationResult& Result, const TArray<uint8>& Data)
        {
            // Data порожній: байти у файлі.
        }));

// У пам'ять.
Client->DownloadBytes(TEXT("my-bucket"), TEXT("config.json"),
    FS3OnDownloadResult::CreateLambda(
        [](const FS3OperationResult& Result, const TArray<uint8>& Data)
        {
            if (Result.IsSuccess())
            {
                FString Json;
                FFileHelper::BufferToString(Json, Data.GetData(), Data.Num());
            }
        }));

// Частина об'єкта.
Client->DownloadRange(TEXT("my-bucket"), TEXT("video.mp4"), 0, 1023,
    FS3OnRangeResult::CreateLambda(
        [](const FS3RangeDownloadResult& Result)
        {
            // Result.TotalObjectSize — повний розмір, навіть якщо взяли кілограм байтів.
        }));

// Серією діапазонних запитів, а не одним стрімом - і тому єдине зчитування, яке можна
// продовжити з місця обриву: те, що встигло прийти, лишається у <файл>.s3part.
// 0 бере розмір шматка з налаштувань транспорту; між спробами він може бути іншим.
Client->DownloadFileChunked(TEXT("my-bucket"), TEXT("patches/1.2.pak"), LocalPath,
    /*ChunkSizeBytes=*/0,
    FS3OnDownloadResult::CreateLambda(
        [](const FS3OperationResult& Result, const TArray<uint8>& Data)
        {
            // ES3Result::PreconditionFailed - об'єкт переписали під час зчитування, тож
            // склеїти дві версії плагін відмовився. Зчитайте його наново.
        }));
```

---

## Перелік і пагінація

```cpp
void ListPage(US3Client* Client, const FString& Token)
{
    Client->ListObjects(TEXT("my-bucket"), TEXT("saves/"), TEXT("/"), 1000, Token,
        FS3OnListObjectsResult::CreateLambda(
            [Client](const FS3ListObjectsResult& Result)
            {
                if (!Result.IsSuccess())
                {
                    UE_LOG(LogTemp, Error, TEXT("%s"), *Result.ToLogString());
                    return;
                }

                for (const FString& Folder : Result.CommonPrefixes) { /* «теки» */ }
                for (const FS3Object& Object : Result.Objects)      { /* файли */ }

                if (Result.bIsTruncated)
                {
                    ListPage(Client, Result.NextContinuationToken);
                }
            }));
}

// Перелік бакетів облікового запису. Лише для Amazon у глобальній формі: R2 відповідає
// на цей виклик кодом 403.
Client->ListBuckets(
    FS3OnListBucketsResult::CreateLambda(
        [](const FS3ListBucketsResult& Result)
        {
            for (const FS3Bucket& Bucket : Result.Buckets) { /* Bucket.Name, Bucket.CreationDate */ }
        }));
```

---

## Метадані та видалення

```cpp
Client->GetMetadata(TEXT("my-bucket"), TEXT("save.sav"),
    FS3OnMetadataResult::CreateLambda(
        [](const FS3MetadataResult& Result)
        {
            // Result.ContentLength, ContentType, ETag, LastModified
            // Result.Metadata — власні метадані, ключі у нижньому регістрі
        }));

// Заміна метаданих: копія об'єкта самого в себе на боці провайдера.
TMap<FString, FString> NewMetadata;
NewMetadata.Add(TEXT("game-version"), TEXT("1.4.0"));

Client->SetMetadata(TEXT("my-bucket"), TEXT("save.sav"), NewMetadata, TEXT("application/octet-stream"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));

// Пакетне видалення: по тисячі ключів за запит.
Client->DeleteObjects(TEXT("my-bucket"), MoveTemp(Keys),
    FS3OnBatchDeleteResult::CreateLambda(
        [](const FS3BatchDeleteResult& Result)
        {
            // Може бути PartialSuccess: запит пройшов, а частину ключів відхилено.
            for (const FS3DeletedObject& Item : Result.Results)
            {
                if (!Item.bDeleted)
                {
                    UE_LOG(LogTemp, Warning, TEXT("%s: %s"), *Item.Key, *Item.ErrorMessage);
                }
            }
        }));
```

---

## Копіювання об'єкта

Копіює об'єкт цілком на боці провайдера — байти не проходять через клієнта. Джерело й
призначення можуть бути в різних бакетах.

```cpp
Client->CopyObject(
    TEXT("my-bucket"), TEXT("uploads/screenshot.png"),        // джерело
    TEXT("my-bucket"), TEXT("archive/2026/screenshot.png"),   // призначення
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

`SetMetadata` вище користується тим самим механізмом провайдера — копією об'єкта самого в
себе — щоб замінити метадані без повторного надсилання вмісту.

---

## Теги об'єктів

Пари «ключ-значення» біля об'єкта, які, на відміну від метаданих, можна змінювати будь-коли
без переписування самого об'єкта і без зміни часу останньої зміни. Правила життєвого циклу й
політики доступу можуть відбирати об'єкти саме за тегами — метаданих вони не бачать. Різницю
між тегами й метаданими докладно розібрано в
[«Теги чи метадані: що з них брати»](04-Blueprint-Operations.md#теги-чи-метадані-що-з-них-брати).

```cpp
Client->GetObjectTags(TEXT("my-bucket"), TEXT("uploads/screenshot.png"),
    FS3OnTagsResult::CreateLambda(
        [](const FS3TagsResult& Result)
        {
            // Result.Tags — порожня мапа на успіху означає «тегів просто немає».
        }));

// Це заміна, а не злиття: усе, чого немає в переданій мапі, зникає.
TMap<FString, FString> Tags;
Tags.Add(TEXT("moderation"), TEXT("pending"));

Client->SetObjectTags(TEXT("my-bucket"), TEXT("uploads/screenshot.png"), Tags,
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));

Client->DeleteObjectTags(TEXT("my-bucket"), TEXT("uploads/screenshot.png"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

Значення можуть містити амперсанди, лапки чи кирилицю без додаткового екранування з вашого
боку — плагін сам дбає про XML-документ запиту й відповіді.

---

## Бакети та правила життєвого циклу

```cpp
// Потрібно лише на провайдерах, які не створюють бакет на першому записі в нього.
Client->CreateBucket(TEXT("my-game-saves"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

Правило життєвого циклу — інструкція, яку бакет виконує сам, за розкладом провайдера
(зазвичай раз на добу), без жодного виклику з гри чи сервера. Найважливіше практичне
застосування — прибирання частин перерваних багаточастинних завантажень, які плагін
навмисно лишає в бакеті, щоб їх можна було відновити (див.
[5. Передавання файлів](05-Transfers.md#великі-файли) і
[FAQ про відновлення завантажень](11-FAQ.md#чи-можна-відновити-перерване-завантаження)):

```cpp
FS3LifecycleRule SweepIncompleteUploads;
SweepIncompleteUploads.Id                              = TEXT("sweep-uploads");
SweepIncompleteUploads.AbortIncompleteUploadsAfterDays = 7;

Client->SetBucketLifecycle(TEXT("my-game-saves"), { SweepIncompleteUploads },
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

`SetBucketLifecycle` замінює **весь** набір правил бакета — щоб додати одне до наявних,
спершу прочитайте їх через `GetBucketLifecycle`. Порожній масив за задумом не має сенсу як
«нічого не міняти»: щоб прибрати всі правила свідомо, викличте `DeleteBucketLifecycle`
окремо.

```cpp
Client->GetBucketLifecycle(TEXT("my-game-saves"),
    FS3OnLifecycleResult::CreateLambda(
        [](const FS3LifecycleResult& Result)
        {
            // Result.Rules — порожній масив на успіху означає «правил немає», не помилку.
        }));

Client->DeleteBucketLifecycle(TEXT("my-game-saves"),
    FS3OnResult::CreateLambda([](const FS3OperationResult& Result) {}));
```

Повний опис полів `FS3LifecycleRule` (`Prefix`, `TagFilters`, `ExpireAfterDays`) — у
[«Правила життєвого циклу»](04-Blueprint-Operations.md#правила-життєвого-циклу).

---

## Скасування

```cpp
FS3TransferHandleRef Transfer = Client->DownloadFile(...);

// Пізніше:
Transfer->Cancel();

// Або все відразу:
Client->CancelAllTransfers();
```

Дескриптор дає ще `IsFinished()`, `GetState()` і `GetProgress()`. Тримати його після
завершення нешкідливо.

---

## Підписані посилання

```cpp
FS3PresignedUrlResult Url = Client->GeneratePresignedUrl(
    TEXT("my-bucket"), TEXT("uploads/screenshot.png"),
    ES3HttpMethod::PUT, 900);

if (Url.IsSuccess())
{
    // Віддати клієнтові; він завантажить файл без жодних ключів.
}
```

Обчислюється локально, без запиту до провайдера.

---

## Облікові дані на конкретному клієнті

Найкоротший спосіб віддати клієнтові ключі, які щойно повернув ваш бекенд:

```cpp
Client->SetStaticCredentials(
    Response.AccessKeyId,
    Response.SecretAccessKey,
    Response.SessionToken,
    /*ExpiresInSeconds=*/3600);   // 0 - ключі без терміну дії
```

Діє на той клієнт, на якому викликана, — зокрема на клієнта профілю з
`Get S3 Client For Profile`. Цим вона відрізняється від `US3Subsystem::SetRuntimeCredentials`,
яка налаштовує тільки клієнта за замовчуванням. Доступна і з Blueprint, під тим самим іменем.

Ключі тримаються лише в пам'яті; плагін припиняє ними користуватися трохи раніше за вказаний
строк, щоб запит, підписаний під самий кінець, не прибув уже після його завершення.

---

## Власний провайдер облікових даних

Коли ключі мають оновлюватися самі, а не встановлюватися один раз — для клієнта гри, який
отримує короткоживучі ключі від вашого бекенда:

```cpp
#include "Auth/S3CredentialsProviders.h"

auto Provider = MakeShared<FS3CredentialsProvider_Callback, ESPMode::ThreadSafe>(
    FS3CredentialsProvider_Callback::FS3CredentialsFetch::CreateLambda(
        [](FS3CredentialsResolved OnFetched)
        {
            MyBackend::RequestS3Credentials(
                [OnFetched](bool bOk, const FStsResponse& Response)
                {
                    OnFetched.ExecuteIfBound(bOk, FS3Credentials(
                        Response.AccessKeyId,
                        Response.SecretAccessKey,
                        Response.SessionToken,
                        Response.ExpiresAtUtc));
                });
        }));

Client->SetCredentialsProvider(Provider);
```

Провайдер оновлює ключі перед закінченням строку й склеює одночасні запити: двадцять
передавань на протермінованих ключах дадуть вашому бекендові один запит.

Готові реалізації: `_Static`, `_Environment`, `_LocalUserStore`, `_Callback`, `_Anonymous`
і `FS3CredentialsProviderChain`, який перебирає їх по черзі.

---

## Підміна транспорту для тестів

```cpp
Client->SetHttpTransport(MyFakeTransport);
```

Саме цей шов дозволяє перевіряти повтори, послідовність багаточастинного завантаження й
скасування без мережі. Докладніше — у розділі [Тестування](09-Testing.md).

---

**Далі:** [7. Помилки та діагностика](07-Errors-And-Diagnostics.md)
