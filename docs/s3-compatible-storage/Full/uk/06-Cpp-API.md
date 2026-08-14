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

## Власний провайдер облікових даних

Для клієнта гри, який отримує короткоживучі ключі від вашого бекенда:

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
