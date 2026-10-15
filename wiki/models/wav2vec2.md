---
type: model
tags: [ssl, transformer, asr, frontend]
sources: [pretrained-models-review, nb-voxceleb-real-audio-benchmark]
status: untested
updated: 2026-08-27
---

# wav2vec2

| | |
| :-- | :-- |
| Модель | `Saire2023/wav2vec2-base-finetuned-Speaker-Classification` |
| Оригінальна стаття | [arXiv:2006.11477](https://arxiv.org/abs/2006.11477) |
| Ембединг | **768** (найбільший серед кандидатів) |
| Тренувальні дані | LibriSpeech + fine-tuning |
| Архітектура | Transformer (SSL) |
| Заявлений EER | **—** (не вказано в джерелі) |

Модель, **розроблена для транскрибування** і дотренована під ідентифікацію за голосом.
Тобто ідентифікація мовця — не її рідна задача, а адаптація.

## Використання в проєкті

`facebook/wav2vec2-base` — фронтенд у ноутбуці
[[sources/nb-voxceleb-real-audio-benchmark]] (768-dim → mean-pooling → MLP-голова).
Але там це **бінарний детектор діпфейків**, а не енкодер мовця. Спільне тільки те,
що обидві задачі знімають ознаки з того самого SSL-бекбону.

Практичні деталі звідти: при завантаженні `Wav2Vec2Model` попереджає про UNEXPECTED-ключі
(`project_hid`, `quantizer.*`, `project_q`) — це нормально, це голови претренувальної задачі,
не потрібні для витягу ознак.

## Оцінка для проєкту

**Найнижчий пріоритет.** Немає жодної метрики, задача неоригінальна, а як SSL-фронтенд
[[models/wavlm]] явно сильніший (WavLM тренувався зокрема під задачі, чутливі до мовця).

## Джерела

[[sources/pretrained-models-review]] · [[sources/nb-voxceleb-real-audio-benchmark]]
