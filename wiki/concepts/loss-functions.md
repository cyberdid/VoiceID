---
type: concept
tags: [loss, aam-softmax, margin, тренування]
sources: [wespeaker-toolkit, voxwatch-osi-benchmark]
updated: 2026-08-27
---

# Функції втрат і margin-розклади

Заповнює прогалину №15 з [[open-questions]]: [[sources/voice-id-parameters-methods]]
описує архітектури, але жодного слова про те, **за яким критерієм** їх навчають —
хоча саме це визначає якість ембедингів.

## Ієрархія

| Loss | Ідея | Де в проєкті |
| :-- | :-- | :-- |
| Softmax cross-entropy | звичайна класифікація по дикторах | базовий варіант |
| **A-softmax** | кутовий margin (множинний) | підтримується Wespeaker |
| **AM-softmax** | адитивний margin у косинусі | підтримується Wespeaker |
| **AAM-softmax** (additive angular margin, ArcFace) | адитивний **кутовий** margin | **стандарт**: ResNet-124 і WavLM-TDNN у VoxWatch, усі рецепти Wespeaker |
| Angular prototypical | метричний, попарний | ResNet-34 (clovaai) у VoxWatch |
| GE2E | метричний, попарний | [[models/resemblyzer]] |

**Навіщо margin.** Звичайний softmax лише розділяє класи. Margin-лосси змушують класи
бути розділеними **із запасом** — ембединги одного диктора стискаються, різних —
розсуваються. Наслідок практичний: після навчання з large-margin лоссом
**звичайного cosine similarity достатньо** як бекенда, PLDA більше не потрібна
([[sources/wespeaker-toolkit]]).

## Margin не ставлять одразу — його нарощують

Wespeaker використовує **триетапний розклад** `m(t)`:

```
m(t) = 0        при t < T₁          (спочатку — звичайний softmax)
     = f(t)     при T₁ ≤ t < T₂     (лінійне або логарифмічне зростання 0 → M)
     = M        при T₂ ≤ t < T      (фіксований фінальний margin)
```

Причина: з великим margin від початку модель не сходиться.

Паралельно розклад LR: `lr(t) = g(t)·h(t)`, де `g` — warmup (лінійний до `T_warm`),
`h` — експоненційне спадання від η₀ до η_T.

## Large margin fine-tuning

Окремий **фінальний етап** поверх уже навченої моделі: **більший margin + довші сегменти**.

| Джерело | Рецепт фінального етапу |
| :-- | :-- |
| Wespeaker | AAM **margin 0,5**, сегменти **6 с** |
| VoxWatch, ResNet-124 | етап 1: 2 с / margin 0,2 → етап 2: **6 с / margin 0,5** |
| VoxWatch, WavLM-TDNN | етапи 1–2: 2–3 с / margin 0,2 → етап 3: **6 с / margin 0,4** |

Три незалежні системи збіглися на «6 секунд + margin 0,4–0,5». Це де-факто стандарт.

Саме цей етап дає суфікс **`-LM`** у назві `wespeaker-voxceleb-resnet34-LM`
([[models/resnet]], [[models/wespeaker]]).

⚠️ Нюанс: на етапі large-margin fine-tuning **speed perturbation вимикають**
([[sources/voxwatch-osi-benchmark]]).

## Чому це важливо для нас

1. Якщо дійде до дотреновування ([[concepts/fine-tuning]]) — **брати AAM-softmax**,
   не звичайний softmax і не BCE. (У ноутбуці
   [[sources/nb-voxceleb-real-audio-benchmark]] використано BCE — але там бінарна задача,
   не класифікація дикторів, тому це коректно саме там.)
2. **Нарощувати margin і довжину сегментів**, а не ставити фінальні значення одразу.
3. Різниця «оригінальна ResNet34 1,31% vs Wespeaker ResNet34 0,723%» — це переважно
   різниця тренувального рецепту, зокрема margin-розкладу і large-margin етапу.
   **Рецепт важить стільки ж, скільки архітектура.**

## Пов'язані

[[concepts/fine-tuning]] · [[concepts/speaker-embedding]] · [[concepts/score-normalization]] ·
[[sources/wespeaker-toolkit]]
