# Посібник користувача Fuzzy Logic

*[🇬🇧 English](../en/README.md) | 🇺🇦 Українська*

**Fuzzy Logic** — плагін нечіткого логічного висновку Мамдані для Unreal Engine. Він дає змогу описувати поведінку зрозумілими правилами `IF / THEN`, плавно поєднувати кілька умов і отримувати одне або кілька числових рішень без каскадів жорстких порогів.

Системи зберігаються як повторно використовувані `UFuzzySystemAsset`, можуть працювати безпосередньо в `UFuzzyLogicComponent`, імпортуються й експортуються у JSON, перевіряються редактором і візуалізуються у Slate та UMG.

> **Fuzzy Logic 1.0**, для Unreal Engine 5.8. Що нового в цьому випуску — [Примітки до випуску](Release-Notes.md).

## Швидкий старт

1. Створіть **Fuzzy System** через **Content Browser → Add → Miscellaneous → Data Asset**.
2. Додайте вхідну змінну `Distance` з діапазоном `0…2000` і множинами `Near`, `Mid`, `Far`.
3. Додайте вихід `Aggression` з діапазоном `0…1` і множинами `Cautious`, `Bold`.
4. Запишіть правила:

   ```text
   IF Distance IS Near THEN Aggression IS Cautious
   IF Distance IS Far THEN Aggression IS Bold
   ```

5. Відкрийте Data Asset подвійним кліком. На вкладці **Inference** змінюйте `Distance` зліва й одразу дивіться число та графік `Aggression` справа.
6. Додайте компонент **Fuzzy Logic** до актора, призначте створений `System Asset`, викличте `Set Input`, а потім `Evaluate Output`.

## Зміст

- [Примітки до випуску](Release-Notes.md) — що нового, що виправлено та на що звернути увагу під час оновлення.
1. [Вступ](01-Introduction.md) — навіщо потрібна нечітка логіка та як читати її результати.
2. [Перші кроки](02-Quick-Start.md) — мінімальна робоча система від Data Asset до Blueprint.
3. [Основні поняття](03-Core-Concepts.md) — змінні, множини, правила, результати й поверхні.
4. [Функції належності](04-Membership-Functions.md) — дев’ять вбудованих форм та вибір параметрів.
5. [Авторинг системи](05-Authoring-A-System.md) — побудова стійкої бази правил у Data Asset.
6. [Мова правил](06-Rule-Language.md) — повний синтаксис `IF / THEN`.
7. [Логічний висновок і налаштування](07-Inference-And-Settings.md) — норми, імплікація, агрегація та дефазифікація.
8. [Редактор Data Asset](08-Asset-Editor.md) — Details, Variables, Inference, Compile Log і JSON-команди.
9. [Інтеграція у Blueprint](09-Blueprint-Integration.md) — компонент, бібліотека та структура результату.
10. [Інтеграція на C++](10-CPP-Integration.md) — модулі, рушій висновку та розширення.
11. [JSON-пресети](11-JSON-Presets.md) — двонапрямний обмін, схема й API.
12. [Діагностика та валідація](12-Diagnostics-And-Validation.md) — пошук помилок у системі й правилах.
13. [Візуалізація в UMG](13-UMG-Visualization.md) — побудова графіків у грі.
14. [Архітектура та продуктивність](14-Architecture-And-Performance.md) — кешування, потоки й вартість обчислень.
15. [Поширені запитання](15-FAQ.md) — короткі відповіді на типові проблеми.
16. [Демонстраційний контент](16-Demo-Content.md) — карти, окремий модуль і виключення Demo зі збірки.
17. [Демо Fuzzy Drone Arena](17-Drone-Arena-Demo.md) — готовий приклад із дев’ятьма агентами.
18. [Нечіткі системи Fuzzy Turret Defense](18-Turret-Defense-Fuzzy-Glossary.md) — словник змінних, множин і порогів демонстрації турелі.
