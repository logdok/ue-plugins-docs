# Вузли та вільні вузли (Knobs)

*[🇬🇧 English](../en/05-Knobs-And-Free-Knobs.md) | 🇺🇦 Українська*

> ⚠️ **Це видання SplineCraft Demo (оцінювальне).** Кожна можливість працює точно так само, як у повному плагіні; обмежено лише *обсяг* того, що можна побудувати, показано watermark на екрані, і в Shipping-збірках воно нічого не робить. Повний перелік: [Обмеження демо](00-Demo-Limitations.md).

**Вузли** — декоративні меші або актори: навершя стовпів, кулі, шпилі, ліхтарі, вивіски.
У SplineCraft Demo є три різновиди.

## 1. Секція KNOBS

Секція **KNOBS** на акторі влаштована так само, як [POSTS](03-Posts.md): масив `Knobs`, кожен
запис — це `FDemoSCPoint`, який ставить меш у кожній активній точці сплайна. Різниця лише
семантична — цю секцію зручно тримати окремо для суто декоративного шару.

- **Visible All**, **Visible All Knobs**
- **All Knobs Visibility Configuration** — правила видимості для всіх записів секції
- кожен `FDemoSCPoint` має власний масив **Point Knobs** — вузли, змонтовані на ньому

### Вузол на точці (`FDemoSCPointKnob`)

| Параметр | Призначення |
|---|---|
| **Distance From Owner** | зсув від власника вздовж його осі вгору (см). |
| **Main Static Mesh Configuration** (`FDemoSCPointKnobMeshConfig`) | **Scale**, **Shift**, **Rotation** (усі `FVector` / `FRotator`), **Static Mesh**, **Displayed Object**, **Actor**. |
| **Alternate / Special / Random Meshes** | підміна за позицією, за діапазонами індексів, випадково. |
| **Detailed Visibility Configuration** | правила видимості — див. [розділ 7](07-Visibility-Rules.md). |
| **Ignore … Randomization** / **Randomization Settings** | розкид масштабу та кутів. |

## 2. Вузли на секціях і трубах

Задаються прямо в елементі (`FDemoSCElementKnob`) і кріпляться до прольоту у ваговій позиції
**Weight Horizontal Position** (0..1). Опис — у розділі
[Секції та труби](04-Sections-And-Tubes.md).

## 3. Вільні вузли (секція FREE KNOBS)

`FDemoSCFreePointKnobs` / `FDemoSCFreePoint` — вузли, **не прив'язані до точок сплайна**. Ставляться
в локальному просторі актора: положення визначає **Distance From Bottom** (підйом по Z) плюс
**Shift** із конфігурації меша. Годяться для поодиноких об'єктів — таблички, ліхтаря,
одиничної деталі біля структури.

| Параметр | Призначення |
|---|---|
| **Visible** | показувати вузол. |
| **Distance From Bottom** | підйом по Z у локальних координатах актора (см). |
| **Main Static Mesh Configuration** (`FDemoSCPointMeshConfig`) | Scale, Shift, Rotation, Static Mesh, Displayed Object, Actor. |
| **Use Random Meshes** + **Random Meshes** | випадковий вибір меша. |

`FDemoSCFreePointKnobs` має спільний вимикач **Visible All**.

## Меш чи актор (`EDemoShowKnobMode`)

- **Mesh** — показати статичний меш. Видно і в редакторі, і у грі.
- **Actor** — заспавнити актор заданого класу. Видно **лише у грі**. Значення `Shift` і
  `Rotation` беруться з налаштувань вузла. Детально — [Додаткові актори](12-Additional-Actors.md).

## Дивіться також

- [Стовпи (Posts)](03-Posts.md)
- [Секції та труби](04-Sections-And-Tubes.md)
- [Рандомізація та Random Seed](08-Randomization.md)
