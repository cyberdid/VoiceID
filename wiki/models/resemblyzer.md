---
type: model
tags: [lstm, ge2e, легка-імплементація]
sources: [pretrained-models-review]
status: candidate
updated: 2026-08-27
---

# Resemblyzer

| | |
| :-- | :-- |
| Модель | [github.com/resemble-ai/Resemblyzer](https://github.com/resemble-ai/Resemblyzer) |
| Стаття | [arXiv:1710.10467](https://arxiv.org/pdf/1710.10467) (GE2E loss) |
| Ембединг | **256** |
| Тренувальні дані | VoxCeleb1 + VoxCeleb2 + LibriSpeech train (36 М висловлювань, 18 тис. дикторів) |
| Архітектура | **GE2E LSTM** |
| Заявлений EER | 3,55% |

Найстарша архітектура в списку (LSTM, 2017) — до епохи TDNN/ResNet-енкодерів.

## Оцінка для проєкту

**Не як основна модель** — точність низька. Але «легка в імплементації» і має найбільшу
базу дикторів у тренуванні (18 тис.).

Реальна ніша: **швидкий санітарний baseline за 10 хвилин**. Якщо на наших даних Resemblyzer
дає осмислені результати — пайплайн і дані в порядку; якщо ні — проблема в даних, а не в моделі.
На українських даних не тестувалась.

## Джерела

[[sources/pretrained-models-review]]
