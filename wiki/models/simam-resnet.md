---
type: model
tags: [simam, resnet, voxblink2, multilingual, кандидат-1]
sources: [lit-wespeaker-repo, lit-model-checkpoints]
status: candidate
confidence: "Числа з офіційного README Wespeaker; ваги не завантажувались"
updated: 2026-08-27
---

# SimAM_ResNet (VoxBlink2)

**Найсильніший кандидат проєкту за опублікованими числами.** ResNet із вбудованим
**SimAM** — простим attention-модулем **без додаткових параметрів** (plug-and-play),
попередньо натренований на [[datasets/voxblink2]] і дотренований на VoxCeleb2.

## Числа

З LM fine-tuning + AS-Norm + QMF ([[sources/lit-wespeaker-repo]]):

| Модель | vox1-O | vox1-E | vox1-H |
| :-- | --: | --: | --: |
| **SimAM_ResNet100** | **0,202** | **0,421** | **0,795** |
| **SimAM_ResNet34** | **0,372** | **0,559** | **0,997** |
| ResNet293-TSTP (лише VoxCeleb2) | 0,425 | 0,641 | 1,146 |
| ResNet34-TSTP (лише VoxCeleb2) | 0,659 | 0,821 | 1,437 |
| ECAPA_TDNN_c1024 | 0,707 | 0,894 | 1,615 |

## 🔴 Головний доказ проєкту

**SimAM_ResNet34 обходить ResNet293.** 0,372 проти 0,425 на Vox1-O, 0,997 проти 1,146
на Vox1-H. **34 шари б'ють 293 шари.**

Різниця не в архітектурі й не в глибині — а в тому, що модель бачила **110 тис.
дикторів замість 6 тис.** Це найчистіше підтвердження тези, що для чужого домену
широта тренувальних даних важить більше за архітектуру.

Порівняння з тією ж SimAM_ResNet34 без VoxBlink2-претренування в таблиці немає, тож
внесок саме претренування виміряний лише опосередковано — але напрям однозначний.

## Чому це кандидат саме для нас

**Це єдина лінійка Wespeaker, позначена як `Multilingual`** — VoxCeleb-моделі там
`English`, CNCeleb-моделі `CN`. Для російськомовного [[datasets/telegram-corpus]]
це вибір за визначенням, а не за здогадкою.

## Як брати

HuggingFace: **`Wespeaker/wespeaker-voxlink2-samresnet34_ft`**
⚠️ **опечатка в назві репозиторію** — `voxlink2`, не `voxblink2`. Пошук по «voxblink»
його не знаходить, натомість видає сторонні перезаливки без model card — їх не брати.

Офіційні ваги також на [Google Drive](https://drive.google.com/drive/folders/19uUmlnSjmFhffO7vHXO7upZJI77OOmrP)
проєкту VoxBlink2. Ліцензія: **CC BY-NC-SA 4.0, некомерційна**.

Підтримка в Wespeaker з **2024.09.03**.

## ❌ Що невідомо

**Поведінка на коротких записах не опублікована.** Ані в README, ані в статті
VoxBlink2 — жодних 2 с / 3 с чисел. Для [[models/3d-speaker]] ERes2NetV2 такі числа є,
для SimAM_ResNet — ні.

Це критично, бо голосові Telegram короткі ([[concepts/short-duration]]). І це саме
та невідома, через яку **не можна обрати між SimAM_ResNet і ERes2NetV2 з паперу**.

**Закривається самостійним заміром за кілька GPU-годин**: обрізати Vox1-O до 2 с і 3 с,
прогнати обидві моделі. Входить в етап P0 [[experiments/clustering-pilot]].

## Позиція в шортлісті

Кандидат №1 або №2 — нарівні з [[models/3d-speaker]] ERes2NetV2:

| | SimAM_ResNet (VoxBlink2) | ERes2NetV2 (ModelScope) |
| :-- | :-- | :-- |
| Мова тренування | **multilingual** (YouTube, 110 тис. дикторів) | китайськоцентрична (200 тис. китайських) |
| Vox1-O | **0,202–0,372** | 0,61 |
| Чужий домен | — (немає CNCeleb-числа) | **6,14% CNCeleb — найкраща** |
| Короткі записи | **невідомо** | **1,48% @2с — виміряно** |
| Ліцензія | CC BY-NC-SA (некомерційна) | перевірити |

**Вони виграють за різними критеріями, і жоден не має повного набору чисел.**
Тому в пілоті беруться обидві — вони ще й архітектурно різні, отже придатні для
fusion і для безрозміточної перевірки згодою ([[concepts/speaker-clustering]]).

## Джерела

[[sources/lit-wespeaker-repo]] · [[sources/lit-model-checkpoints]]
