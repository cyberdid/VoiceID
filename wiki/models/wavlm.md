---
type: model
tags: [ssl, transformer, microsoft, frontend]
sources: [pretrained-models-review, voxwatch-osi-benchmark]
status: candidate
updated: 2026-08-27
---

# WavLM (сімейство)

Self-supervised трансформер від Microsoft. Стаття: [arXiv:2110.13900](https://arxiv.org/abs/2110.13900),
[github](https://github.com/microsoft/unilm/tree/master/wavlm).

| Варіант | Ембединг | Тренувальні дані | Параметрів | Заявлений EER |
| :-- | :-- | :-- | :-- | :-- |
| `microsoft/wavlm-base-sv` | 512 | LibriSpeech 960 год | 94,70 М | 4,69% |
| `microsoft/wavlm-base-plus-sv` | 512 | 94k год: Libri-Light 60k + GigaSpeech 10k + VoxPopuli 24k | 94,70 М | 4,26% |
| `microsoft/wavlm-large` | 512 | те саме, 94k год | **316,62 М** | 4,04% |

Патерн видно чітко: більше даних (base → base+) дає −0,43 п.п., більше параметрів
(base+ → large) — ще −0,22 п.п.

## Головне застереження — і головна суперечність проєкту

> ⚠️ **Ці 4% EER — не «стеля WavLM».**
> [[sources/pretrained-models-review]] подає WavLM як найслабше сімейство в таблиці
> (4,04–4,69% проти 0,66–1,05% в інших). Але у [[sources/voxwatch-osi-benchmark]]
> **WavLM-Large як фронтенд + ECAPA-TDNN (C=512) як бекенд + AAM-loss + триетапне
> навчання дає EER 0,52% на Vox1-O** — тобто в ~8 разів краще і на рівні найкращих систем.
>
> Різниця не в моделі, а в тому, **що саме взято**: готова SV-голова `*-sv` (навчена на
> обмежених даних) проти WavLM у ролі **фронтенда з власно натренованим бекендом**.
>
> Висновок для нас: **WavLM — не готова модель, а фронтенд.** Оцінювати `wavlm-*-sv`
> «як є» — і оцінювати WavLM як основу для дотреновування — це два різні рішення.

Рецепт VoxWatch (триетапне навчання WavLM-TDNN):
1. WavLM заморожено, тренується TDNN — чанки 2–3 с, margin 0,2;
2. TDNN заморожено, донавчається WavLM (крім згорткового енкодера) — малий LR;
3. спільна оптимізація — чанки 6 с, margin 0,4.

Пор. з дуже схожою схемою поступового розморожування в
[[sources/nb-voxceleb-real-audio-benchmark]] — [[concepts/fine-tuning]].

## Практика на українських даних

`wavlm-base` тестувалась, «показувала непогану точність, але нижчу, ніж інші моделі».
В одному експерименті поступилася [[models/titanet]].

## Оцінка для проєкту

- Як **готовий енкодер `*-sv`** — низький пріоритет.
- Як **фронтенд для дотреновування** під українську/телефонну специфіку — **найперспективніший
  варіант**, якщо дійде до реального тренування. Ціна: 316 М параметрів у large.

## Джерела

[[sources/pretrained-models-review]] · [[sources/voxwatch-osi-benchmark]]
