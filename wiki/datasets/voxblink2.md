---
type: dataset
tags: [voxblink2, великий-корпус, open-set, sota]
sources: [lit-model-checkpoints, lit-ssl-clustering, pretrained-models-review]
confidence: "Офіційна сторінка проєкту та docs Wespeaker; ваги не завантажувались"
updated: 2026-08-27
---

# VoxBlink2

[arXiv:2407.11510](https://arxiv.org/abs/2407.11510) (Interspeech 2024) ·
<http://voxblink2.github.io>

| Параметр | Значення |
| :-- | --: |
| Дикторів | **110 000+** |
| Висловлювань | **~10 млн** |
| Тип | аудіо-візуальний, «in the wild» (відео) |
| SOTA одиночної моделі на Vox1-O | **EER 0,170%**, minDCF 0,006% |

## Чому це важливо

**Порядок різниці в масштабі.** VoxCeleb2-dev — 5 994 диктори. VoxBlink2 — 110 тис.,
у **18 разів більше**. А кількість дикторів у тренуванні — саме той параметр, який
визначає, наскільки енкодер узагальнюється на **незнайомих** людей і незнайомі мови.

**І в якості.** 0,170% проти 0,723% у найкращої моделі нашого каталогу
([[models/comparison]]) — приблизно **вчетверо краще**. Архітектури — ResNet різних
розмірів з ASP та TSP-пулінгом, тобто не екзотика: різниця майже цілком у даних.

## Власний Open-Set benchmark

Автори вводять задачу **Open-Set Speaker-Identification**: зіставити тестовий запис
із відомим мовцем із галереї **або** позначити як невідомого. Це саме та постановка,
що у [[sources/voxwatch-osi-benchmark]], і другий незалежний бенчмарк для неї.

Актуально для **наступного** етапу проєкту — коли з кластерів будуватиметься watchlist
([[concepts/open-set-identification]]).

## Значення для нашого корпусу

[[datasets/telegram-corpus]]: мова російська, кількість мовців невідома, домен чужий
для всіх претренованих моделей. У такій ситуації **широта тренувальних даних важить
більше за архітектуру** — а 110 тис. дикторів «у дикій природі» це якраз широта.

Гіпотеза: чекпойнт, тренований на VoxBlink2, перенесеться на Telegram-російську
краще, ніж чисто-VoxCeleb'івський. *(не перевірено — потребує заміру на пілоті)*

Побічно це вже підтверджувалось: [[sources/pretrained-models-review]] згадує, що
частина чекпойнтів [[models/wespeaker]] тренована саме на **VoxBlink2** і CNCeleb.

## ✅ Чекпойнти доступні (перевірено 2026-08-27)

Два шляхи ([[sources/lit-model-checkpoints]]):

**1. Офіційний** — [Google Drive](https://drive.google.com/drive/folders/19uUmlnSjmFhffO7vHXO7upZJI77OOmrP),
«моделі різного розміру». **EER 0,228%** без пост-обробки, **0,170%** зі Score-Norm + QMF.

**2. Через Wespeaker** ⭐ — і це важливіше. У власній таксономії Wespeaker
([[models/wespeaker]]) VoxCeleb-моделі позначені як **English**, CNCeleb — як **CN**,
а **VoxBlink2 — як Multilingual**: `SimAMResNet34`, `SimAMResNet100`, плюс варіанти
«VoxBlink2 (pretrain) + VoxCeleb2 (finetune)».

**Для російськомовного корпусу це перший кандидат за визначенням.**

HuggingFace: `Wespeaker/wespeaker-voxlink2-samresnet34_ft`
⚠️ **опечатка в назві репозиторію** — `voxlink2`, не `voxblink2`. Через це він
не знаходиться пошуком по «voxblink»; сторонні перезаливки без model card
(`gaunernst/...`) брати не варто.

## ⚠️ Ліцензія

**CC BY-NC-SA 4.0 — комерційне використання заборонене** (дані з YouTube).
Поширюється і на корпус, і на моделі. Контекст проєкту — дослідження, тож не блокує,
але знати наперед варто.

## Побічне підтвердження щодо калібрації

0,228% → 0,170% = **−25% відносно** від самих лише Score-Norm + QMF, на SOTA-моделі.
Незалежно збігається з −21% у [[sources/voxwatch-osi-benchmark]].
Калібрація працює на будь-якому рівні якості — [[concepts/score-normalization]].

## Що лишилось перевірити

- Які саме архітектури й розміри в Google Drive (сторінка не деталізує).
- Чи є в open-set бенчмарку розміри галереї, зіставні з VoxWatch (5…1210).
- Поведінка SimAMResNet на **коротких** записах — [[concepts/short-duration]].

## Пов'язані

[[sources/lit-ssl-clustering]] · [[datasets/voxceleb]] · [[models/comparison]] ·
[[concepts/open-set-identification]] · [[concepts/domain-mismatch]]
