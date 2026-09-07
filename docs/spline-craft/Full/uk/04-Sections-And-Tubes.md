# Секції та труби (Sections & Tubes)

*[🇬🇧 English](../en/04-Sections-And-Tubes.md) | 🇺🇦 Українська*

**Секції** та **труби** розміщують меш у проміжку між сусідніми активними точками сплайна.
Обидві секції використовують той самий тип конфігурації (`FSCElement`); відмінність — лише
призначення та типові меші: секції — це панелі, заповнення, щити; труби — поручні, рейки,
перекладини.

## Елемент (`FSCElement`)

| Параметр | Призначення |
|---|---|
| **Distance From Bottom** | висота елемента над сплайном (см). |
| **Visible All** | вимикач запису. |
| **Use Every Multiple Of Index** | крок повторення: елемент будується через кожні N точок сплайна. |
| **Main Static Mesh Configuration** | головна конфігурація меша (нижче). |
| **Detailed Visibility Configuration** | детальні правила видимості — див. [Правила видимості](07-Visibility-Rules.md). |
| **Alternate Meshes** | First / Last / Odd / Even — підміна за позицією прольоту. |
| **Special Meshes** | підміна за діапазонами індексів (**Visible At Index Ranges**). |
| **Random Meshes** | `Use Random Meshes` + список; меш обирається випадково. |
| **Knobs** | вузли, змонтовані на елементі (нижче). |
| **Visible Knobs** | вимикач усіх вузлів елемента. |
| **Ignore Alternate / Special Meshes** | вимкнути відповідний механізм підміни. |

## Головна конфігурація меша (`FSCElementMeshConfig`)

| Параметр | Призначення |
|---|---|
| **Static Mesh** | меш прольоту. |
| **Visible** | показувати меш. |
| **Scale** | **Thickness (Y)** та **Height (Z)** — вагові коефіцієнти від початкового розміру. |
| **Paddings** | відступи (`FMargin`, см): Left / Right коротшають проліт уздовж сплайна, Top / Bottom підрізають по висоті. |
| **Shift** | **Horizontal (Y)** / **Vertical (Z)** зсув (см). |
| **Roll Angle (X)** | нахил навколо осі елемента (градуси). |
| **Mode** (`ESplineCraftMode`) | `Line` або `Curve` для цього конкретного меша — застосовується, коли глобальний **Mode** актора = `Concrete`. Див. [Режими Line та Curve](09-Line-vs-Curve-Mode.md). |
| **Ignore … Randomization Settings** | окремо вимкнути рандомізацію масштабу / відступів / зсуву / повороту. |
| **Randomization Settings** (`FSCElementRandomSettings`) | розкид — див. [Рандомізація](08-Randomization.md). |

**Як меш заповнює проміжок.** У режимі `Line` меш розтягується вздовж осі X так, щоб зайняти
відстань між точками (за вирахуванням Left / Right paddings), а по Z масштабується з
урахуванням Top / Bottom paddings. У режимі `Curve` меш згинається вздовж кривої
`SplineMeshComponent`, а зсув задається через його зміщення початку/кінця.

## Вузли на елементі (`FSCElementKnob`)

Декоративний меш або актор, прикріплений до прольоту у ваговій позиції:

| Параметр | Призначення |
|---|---|
| **Weight Horizontal Position** | позиція вздовж прольоту, 0..1. |
| **Shift** (`FSCShift`) | зсув Horizontal / Vertical (см). |
| **Scale** | масштаб (`FVector`). |
| **Rotation** | доданий поворот. |
| **Static Mesh** / **Displayed Object** / **Actor** | що показувати (`Mesh` або `Actor`). |
| **Ignore … Randomization** / **Randomization Settings** | розкид на рівні вузла. |

Вузли будуються в обох режимах — Line і Curve. Разом із прихованим елементом ховаються і його
вузли.

## Видимість на рівні секції

`FSCSections` / `FSCTubes`:

- **Visible All Sections** / **Visible All Tubes** — вимикач усієї секції.
- **Visible All Knobs** — вимикач вузлів на всіх елементах.
- **All Sections/Tubes Visibility Configuration** — правила видимості для всіх елементів одразу.

## Дивіться також

- [Режими Line та Curve](09-Line-vs-Curve-Mode.md)
- [Вузли та вільні вузли](05-Knobs-And-Free-Knobs.md)
- [Правила видимості та проєми](07-Visibility-Rules.md)
