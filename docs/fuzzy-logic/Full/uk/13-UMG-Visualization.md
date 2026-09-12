# 13 — Візуалізація в UMG

*[🇬🇧 English](../en/13-UMG-Visualization.md) | 🇺🇦 Українська*

Модуль `FuzzyLogicUMG` малює функції належності та результати у `FPaintContext`. Викликайте вузли з події **On Paint** вашого `UserWidget`.

## Область графіка

`FFuzzyPlotArea` містить:

| Поле | Значення |
|---|---|
| `Origin` | Верхній лівий кут у координатах віджета |
| `Size` | Ширина й висота області |
| `Range` | Значення горизонтальної осі |

Належність `1` розташована на верхньому краї, `0` — на нижньому.

## Blueprint-вузли

| Вузол | Малює |
|---|---|
| **Draw Fuzzy Set** | Одну функцію належності |
| **Draw Fuzzy Variable** | Усі множини змінної кольорами палітри |
| **Draw Fuzzy Surface** | Агреговану поверхню результату |
| **Draw Fuzzy Value Marker** | Вертикальну лінію чіткого значення |
| **Draw Fuzzy Membership Marker** | Горизонтальну лінію ступеня |
| **Draw Fuzzy Plot Axes** | Базову та ліву вісь |
| **Project Fuzzy Point** | Координату точки для власного підпису |
| **Get Default Fuzzy Palette** | Шість контрастних кольорів |

## Графік останнього рішення

1. Викличте `Evaluate` на компоненті.
2. Отримайте `Get Last Output Surface` для потрібного виходу.
3. Передайте поверхню до `Draw Fuzzy Surface`.
4. Візьміть чітке число з `Result.Outputs` і намалюйте `Draw Fuzzy Value Marker`.

Так UI показує ту саму дискретизовану криву, яку використала дефазифікація.

## Приклад On Paint

У Blueprint-події `On Paint`:

```text
Make Fuzzy Plot Area
  Origin = (24, 24)
  Size   = (420, 180)
  Range  = OutputSurface.Range

Draw Fuzzy Plot Axes
Draw Fuzzy Surface
Draw Fuzzy Value Marker
```

Для `Draw Fuzzy Set` і `Draw Fuzzy Variable` параметр `Samples = 129` зазвичай достатній. Збільшуйте його для дуже широкого віджета або крутих функцій.

## Власні форми

Малювання працює через базовий `FFuzzyMembershipFunction`, тому нова C++-форма відображається без окремого графічного коду.

## Weighted Average

Цей метод не використовує агреговану поверхню для обчислення числа. Якщо ви будуєте пояснювальний UI, позначте криву як ілюстративну, так само як це робить редактор Data Asset.
