---
type: model
tags: [resnet, embedding, speechbrain, wespeaker, кандидат]
sources: [pretrained-models-review, voice-id-parameters-methods, voxwatch-osi-benchmark, wespeaker-toolkit]
status: candidate
updated: 2026-08-27
---

# ResNet (сімейство)

Під «ResNet» у джерелах ховаються **щонайменше чотири різні моделі**. Не плутати.

| Варіант | Джерело | Вхід | Ембединг | Loss / pooling | EER |
| :-- | :-- | :-- | :-- | :-- | :-- |
| **SpeechBrain ResNet TDNN** `speechbrain/spkrec-resnet-voxceleb` | [[sources/pretrained-models-review]] | — | **256** | — | 1,05% (заявл.), Vox1+Vox2 |
| **Wespeaker ResNet34-LM** `pyannote/wespeaker-voxceleb-resnet34-LM` | [[sources/pretrained-models-review]] | — | **512** | LM (large margin) | 0,867–1,959% (заявл.) |
| **ResNet-34** (clovaai/voxceleb_trainer, half channels) | [[sources/voxwatch-osi-benchmark]] | 64-dim log-mel | **512** | ASP + angular prototypical, Adam | **1,02%** Vox1-O |
| **ResNet-124** (128 base ch., {6,16,36,3}) | [[sources/voxwatch-osi-benchmark]] | 80-dim log-mel | **256** | mean+std pooling + **AAM**, SGD | **0,41%** Vox1-O |

Згадані також варіанти ResSKNet та ERes2Net (Enhanced Res2Net) як покращення базової
архітектури для верифікації мовця.

## ⭐ Порівняння на єдиному протоколі (Wespeaker)

Це та таблиця, якої бракувало всьому проєкту — усі числа з одного тренування
(Vox2-dev), одного бекенда (cosine + AS-Norm), після large margin fine-tuning.

| Архітектура | Vox1-O | Vox1-E | Vox1-H |
| :-- | --: | --: | --: |
| ECAPA-TDNN (оригінальна) | 0,870 | 1,120 | 2,120 |
| ECAPA-TDNN (Wespeaker) | 0,728 | 0,929 | 1,721 |
| ResNet34 (оригінальна) | 1,31 | 1,38 | 2,50 |
| **ResNet34 (Wespeaker)** | **0,723** | **0,867** | **1,532** |
| ResNet221 | 0,505 | 0,676 | 1,213 |
| ResNet293 | **0,447** | 0,657 | 1,183 |

**ResNet34 і ECAPA-TDNN практично нарівні на Vox1-O (0,723 vs 0,728), але ResNet
виразно кращий на важких умовах** — Vox1-H 1,532 vs 1,721 (−11%). Vox1-H — пари
одного гендеру й національності, тобто найближче до реальної задачі «розрізнити схожі голоси».

Це узгоджується з українським експериментом (ResNet > ECAPA) і **знімає суперечність №16**
з [[open-questions]]: «заявлені EER» порівнювали різні *реалізації*, а не архітектури.

Глибина масштабується: 0,723 → 0,505 → 0,447 (ResNet34 → 221 → 293), що узгоджується
з ResNet-124 (0,41%) у [[sources/voxwatch-osi-benchmark]].

## ⚠️ Реалізація важить не менше за архітектуру

Та сама ResNet34, різні рецепти тренування: **1,31% → 0,723%** (−45%).

І критичніше — **на іншому домені** ([[datasets/cnceleb]]):

| Реалізація ResNet34 | CNCeleb EER |
| :-- | --: |
| **SpeechBrain** | **9,141%** |
| **Wespeaker** | **6,492%** |
| ResNet221 (Wespeaker) | 5,655% |

**−29% відносно** на користь Wespeaker. Оскільки наш домен теж «не VoxCeleb», ця
різниця релевантніша, ніж різниця на Vox1-O.

> 🔁 **Наслідок для шортліста:** кандидат №1 змінюється зі SpeechBrain-чекпойнта на
> **[[models/wespeaker]] ResNet34-LM** (доступний як `pyannote/wespeaker-voxceleb-resnet34-LM`).
> SpeechBrain лишається як контроль — порівняти обидва на своїх даних.

## Ціна зміни домену

Та сама Wespeaker ResNet34: **VoxCeleb 0,723% → CNCeleb 6,492%**. У **9 разів гірше**
лише від зміни мови й типу записів. Головне застереження проєкту —
[[concepts/domain-mismatch]].

## Чому ResNet

Архітектура з комп'ютерного зору, застосована до **спектрограмних представлень**.
Узагальнення з [[sources/voice-id-parameters-methods]]: TDNN сильні в **часовій динаміці**,
ResNet — у **спектральних патернах**. Різні сильні сторони → аргумент за
[[concepts/score-normalization|fusion]].

## Що ми знаємо з практики

- **Найкращий результат в експерименті на великому обсязі україномовних даних (~1,5 год)** —
  кращий за [[ecapa-tdnn]] і [[pyannote]]. Це найсильніший наявний доказ на нашому домені.
- **ResNet-124 — найточніша одиночна система у VoxWatch** (0,41% на Vox1-O), краща за
  WavLM-TDNN. Її рецепт: AAM-loss + двостадійне навчання (2 с/margin 0,2 → 6 с/margin 0,5)
  + аугментація MUSAN/RIR + speed perturbation.
- Wespeaker ResNet34-LM «легка в імплементації», але **на українських даних не тестувалась**.

> ⚠️ **Суперечність.** Заявлений EER SpeechBrain ResNet (1,05%) гірший за ECAPA (0,80%) і
> TitaNet (0,66%), але на українських даних ResNet виграв. Пояснення джерела: заявлені
> цифри міряні на різних тестах. Див. [[concepts/eer-and-metrics]].

## Оцінка для проєкту

**Брати в шортліст, перший на перевірку** — єдиний кандидат з прямим позитивним доказом
на україномовних даних.

## Джерела

[[sources/wespeaker-toolkit]] · [[sources/pretrained-models-review]] ·
[[sources/voxwatch-osi-benchmark]] · [[sources/voice-id-parameters-methods]]
