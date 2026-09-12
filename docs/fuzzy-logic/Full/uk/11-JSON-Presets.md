# 11 — JSON-пресети

*[🇬🇧 English](../en/11-JSON-Presets.md) | 🇺🇦 Українська*

JSON дає змогу зберігати систему поза бінарним Data Asset, переглядати зміни в системі контролю версій, генерувати пресети інструментами та переносити їх між проєктами.

## Data Asset → JSON

Відкрийте `UFuzzySystemAsset` і натисніть **Save To JSON**. Редактор:

1. компілює та перевіряє поточну систему;
2. показує помилки й не записує некоректний документ;
3. відкриває системний діалог збереження;
4. додає `.json`, якщо розширення не задано;
5. записує канонічний документ із `SchemaVersion`.

## JSON → Data Asset

Натисніть **Load Preset** і виберіть `.json`. Завантаження відбувається у тимчасову структуру. Якщо синтаксис, схема або правила некоректні, відкритий Data Asset не змінюється. Після успіху редактор:

- замінює систему;
- оновлює Details, Variables, Inference та Compile Log;
- позначає пакет як змінений;
- створює транзакцію Undo.

Шлях до зовнішнього JSON є станом редактора, а не runtime-властивістю Data Asset. Запакована гра використовує асет і не залежить від loose-файлу.

## Blueprint API

| Вузол | Параметри |
|---|---|
| **Save Fuzzy System To Json** | `System`, `Content Relative Path`, `out Error` |
| **Load Fuzzy System From Json** | `Content Relative Path`, `out System`, `out Error` |

Blueprint-шляхи задаються відносно каталогу `Content`, наприклад `FuzzySystems/Driving.json`. Спроба вийти за межі `Content` через `..` відхиляється. Вбудовані пресети плагіна доступні за повним mount-шляхом, наприклад `/FuzzyLogic/Demo/FuzzyDroneArena/DroneBehavior.json`.

## C++ API

```cpp
#include "Serialization/FuzzySystemJson.h"

FString Json;
FText Error;
FFuzzySystemJson::ToJsonString(System, Json, Error);
FFuzzySystemJson::FromJsonString(Json, System, Error);

FFuzzySystemJson::SaveToFile(System, AbsolutePath, Error);
FFuzzySystemJson::LoadFromFile(AbsolutePath, System, Error);

FString Absolute;
FFuzzySystemJson::ResolveProjectRelativePath(
    TEXT("FuzzySystems/Driving.json"), Absolute, Error);
```

Кожна операція повертає `bool`; `OutError` пояснює невдачу.

## Схема 2

```json
{
  "SchemaVersion": 2,
  "Settings": {
    "AndOperator": "Minimum",
    "OrOperator": "Maximum",
    "Implication": "Clip",
    "Aggregation": "Maximum",
    "DefuzzificationMethod": "Centroid",
    "SampleCount": 201
  },
  "Inputs": [
    {
      "Name": "Distance",
      "Range": { "Min": 0, "Max": 2000 },
      "DefaultValue": 1000,
      "Sets": [
        {
          "Name": "Near",
          "Function": {
            "Type": "Trapezoid",
            "LeftFoot": 0,
            "LeftShoulder": 0,
            "RightShoulder": 300,
            "RightFoot": 700
          }
        }
      ]
    }
  ],
  "Outputs": [
    {
      "Name": "Aggression",
      "Range": { "Min": 0, "Max": 1 },
      "Sets": [
        {
          "Name": "Bold",
          "Function": { "Type": "Triangle", "Left": 0.4, "Peak": 1, "Right": 1 }
        }
      ]
    }
  ],
  "Rules": [
    "IF Distance IS Near THEN Aggression IS Bold"
  ]
}
```

## Поля

- `SchemaVersion` — версія формату. Документ із новішою невідомою версією відхиляється.
- `Settings` — усі математичні налаштування; відсутні поля отримують типові значення.
- `Inputs`, `Outputs` — масиви змінних.
- `Name` і `Range` — обов’язкові для змінної.
- `DefaultValue` — типово `0`.
- `Sets` — масив множин із `Name` та `Function`.
- `Rules` — масив авторського тексту правил.

`Function.Type` приймає `Triangle`, `Trapezoid`, `Gaussian`, `Bell`, `Sigmoid`, `Ramp`, `JShape`, `Singleton`, `Constant` або зареєстрований власний тип. Кожна форма підтримує `"Inverted": true`.

Невідомий тип, пошкоджений або обрізаний JSON завершується чистою помилкою без аварійного завершення редактора.
