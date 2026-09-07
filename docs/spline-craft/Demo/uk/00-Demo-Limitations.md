# Обмеження демо

*[🇬🇧 English](../en/00-Demo-Limitations.md) | 🇺🇦 Українська*

Це **SplineCraft Demo** — оцінювальне видання. Воно встановлюється поряд із повним плагіном,
щоб можна було спробувати кожну можливість перед покупкою. Це **не** урізаний замінник: нічого
не вимкнено й не прибрано, обмежено лише *обсяг* того, що можна побудувати в одному місці.

## Обмежено

| | Демо | Повний |
|---|---|---|
| Точок сплайна на актор | **16** — інструменти розкладки затискають до цього; точки, поставлені вручну понад 16, ігноруються під час збірки | без обмежень |
| Згенерованих компонентів на актор | **250** — instanced-меші + spline-меші + меші полігонів разом; після досягнення решта шарів елементів для цього актора пропускається | без обмежень |
| Watermark на екрані | показується, поки працює рівень (`SplineCraft DEMO — points 16/16, meshes 220/250`) | немає |
| Merge to Static Mesh | **вимкнено** — проєктуйте, дивіться прев'ю та ітеруйте вільно, але запекти постійний Static Mesh-ассет не можна | доступно |
| Shipping-збірки | нічого не генерує, watermark не малює; збірка все одно компілюється й пакується як звичайно | їде з вашою грою |

## Без обмежень — так само, як у повному плагіні

- усі режими розміщення: Concrete / Line / Curve;
- усі інструменти вирівнювання точок: Line, Rectangle, Ellipse, Regular Polygon, Arc, Spiral,
  Sinusoid, Zigzag, Catenary, Bind To Surface, Follow Spline, Uniform, Manual;
- Posts / Sections / Tubes / Knobs / Free Knobs / Polygons / Openings з повною конфігурацією
  масивів / alternate / special / random мешів;
- рандомізація з фіксованим Random Seed;
- Presets та Build Scenarios;
- налаштування колізії та делегати подій зіткнення.

Решта розділів цього посібника застосовна до демо без змін — відрізняються лише п'ять рядків
вище.

## Що постачається з демо

- **`BP_SplineCraftDemo`** (`/SplineCraftDemo/Blueprint/`) — готовий актор, демо-відповідник
  `BP_SplineCraft_PRO` з повного плагіна. Киньте на сцену або поставте **SplineCraft Demo
  Actor** з браузера *Place Actors*.
- Уся бібліотека примітивів — стовпи, секції, труби, вузли — плюс кольорові матеріали й
  текстури, у `/SplineCraftDemo/Primitives`, `.../Material`, `.../Material_Inst`,
  `.../Texture`.
- Приклади у `/SplineCraftDemo/Samples` — `DA_Preset_IronFence`, набір `DA_Preset_Tower_*` та
  `DA_Scenario_GrowingTower`.

## Обидва видання в одному проєкті

Демо (модуль `SplineCraftDemo`, `ADemoSplineCraftActor`, типи `FDemoSC*`) і повний плагін
(модуль `SplineCraft`, `ASplineCraftActor`, типи `FSC*`) — незалежні UObject-типи, їх можна
ввімкнути разом без конфлікту: демо-актор і актор повного плагіна поруч в одному рівні
поводяться кожен сам по собі.

## Перехід на повне видання

Presets та Build Scenarios, створені в демо, — це ассети демо-типів
(`UDemoSplineCraftPreset`, `UDemoSplineCraftBuildScenario`), повний плагін їх не читає —
відтворіть їх там після переходу.

Повне видання на Fab. Повна документація:
https://logdok.github.io/ue-plugins-docs/spline-craft/Full/en/
