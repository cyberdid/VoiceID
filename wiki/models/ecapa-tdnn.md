---
type: model
tags: [tdnn, embedding, speechbrain, кандидат]
sources: [pretrained-models-review, voice-id-parameters-methods, voxwatch-osi-benchmark]
status: candidate
updated: 2026-08-27
---

# ECAPA-TDNN

**Emphasized Channel Attention, Propagation and Aggregation in TDNN.** Один із двох
головних кандидатів проєкту разом з [[resnet]].

| | |
| :-- | :-- |
| Модель | `speechbrain/spkrec-ecapa-voxceleb` |
| Стаття | [arXiv:2005.07143](https://arxiv.org/abs/2005.07143) |
| Ембединг | **192** |
| Тренувальні дані | VoxCeleb |
| Архітектура | Improved TDNN |
| Заявлений EER | **0,80%** |

## Архітектура

Згорткові + residual-блоки, ключові елементи:
- **1D SE-Res2Blocks** (Squeeze–Excitation над Res2Net) — багатомасштабні ознаки + перезважування каналів;
- **attentive statistics pooling** — увага дозволяє зосередитись на найінформативніших
  сегментах аудіо замість рівномірного усереднення;
- модифікована часова динамічна згортка для локальних і глобальних часових ознак.

Тренована на VoxCeleb з аугментацією шумом і реверберацією → стійка до реальних умов запису.

## Що ми знаємо з практики

- **2-ге місце в експерименті на україномовних даних (~1,5 год)**, поступилася лише
  [[resnet]], «досягає майже такої ж точності». Найпоширеніша й найдоступніша модель.
- Легко ставиться через SpeechBrain, є Colab-приклад.
- Використана у [[sources/voxwatch-osi-benchmark]] як бекенд поверх WavLM-Large
  (**WavLM-TDNN**, C=512) → EER **0,52%** на Vox1-O. Тобто сама архітектура ECAPA
  масштабується вгору при сильнішому фронтенді — див. [[wavlm]].

## На єдиному протоколі (Wespeaker)

[[sources/wespeaker-toolkit]] переміряв ECAPA-TDNN у своєму рецепті:

| Реалізація | Vox1-O | Vox1-E | Vox1-H |
| :-- | --: | --: | --: |
| оригінальна | 0,870 | 1,120 | 2,120 |
| **Wespeaker** | **0,728** | 0,929 | 1,721 |
| ResNet34 (Wespeaker), для порівняння | 0,723 | **0,867** | **1,532** |

Два висновки:
- сам лише кращий тренувальний рецепт дає −16% (0,870 → 0,728);
- **на Vox1-O ECAPA і ResNet34 нарівні**, але на важких умовах (Vox1-H, пари одного
  гендеру й національності) **ResNet попереду на 11%**. Це пояснює, чому український
  експеримент дав ResNet > ECAPA. Див. [[models/resnet]].

На CNCeleb ([[datasets/cnceleb]]) ECAPA-TDNN дає 7,395% проти 6,492% у Wespeaker
ResNet34 — той самий порядок переваги ResNet на чужому домені.

## Оцінка для проєкту

**Брати в шортліст.** Найкращий баланс «якість / простота / поширеність». 192-вимірний
ембединг — найкомпактніший серед кандидатів, що здешевлює пошук по великому watchlist.

Ризик: заявлений EER 0,80% міряний не на українських даних — потрібен власний прогін,
[[experiments/protocol]].

## Джерела

[[sources/pretrained-models-review]] · [[sources/voice-id-parameters-methods]] ·
[[sources/voxwatch-osi-benchmark]]
